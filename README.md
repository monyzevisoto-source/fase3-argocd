# Fase 3 — GitOps com Argo CD

Repositório que contém os manifests Kubernetes e as cinco Applications do Argo
CD para executar o ToggleMaster no Amazon EKS.

## Como funciona

Cada Application monitora a branch main deste repositório e aponta para uma
Kustomization independente em services/<serviço>. Quando o GitHub Actions do
repositório fase3-apps publica uma nova imagem, ele atualiza o campo image do
Deployment correspondente e cria um commit.

O Argo CD detecta a diferença entre o Git e o cluster e, como a sincronização
automática está habilitada, aplica a alteração. O Kubernetes baixa a nova imagem
do ECR e realiza um rolling update, validando readiness e liveness probes.

As Applications usam sincronização automática com prune e self-heal habilitados:
alterações manuais no cluster são revertidas e recursos removidos do Git são
apagados do cluster.

## Estrutura

```text
.
├── argocd/
│   ├── kustomization.yaml
│   └── <serviço>.yaml             # Applications dos serviços e observabilidade
└── services/
    └── <serviço>/
        ├── namespace.yaml
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── configmap.yaml
        ├── hpa.yaml               # evaluation e analytics
        └── kustomization.yaml
```

Os serviços são auth-service, flag-service, targeting-service,
evaluation-service e analytics-service. Os Secrets não são versionados neste
repositório.

## Ambientes

Existem dois ambientes: `minikube` e `aws`.

O ambiente `minikube` é executado nesta máquina dentro de um cluster Minikube.
A Application do Argo aponta diretamente para `overlays/minikube`, e as
Applications de observabilidade ficam em `argocd/minikube`.

O ambiente Minikube executa auth-service, flag-service, targeting-service e
evaluation-service com PostgreSQL e Redis dentro do cluster. SQS e DynamoDB não
são usados; analytics-service fica fora deste overlay.

```bash
kubectl apply -k argocd/minikube
kubectl get application togglemaster-minikube -n argocd
```

Para produção, use a Application raiz criada pelo `fase3-terraform`. O
Terraform aponta para `argocd/aws`, que contém as Applications dos serviços e
as variantes AWS de Prometheus, Loki e OpenTelemetry Collector. Não aplique
`argocd/minikube` em um cluster EKS.

## Pré-requisitos

- cluster EKS criado pelo repositório fase3-terraform;
- Argo CD instalado no namespace argocd;
- repositório GitHub acessível pelo Argo CD;
- imagens disponíveis no ECR;
- Secrets criados nos namespaces corretos;
- PostgreSQL, Redis, SQS e DynamoDB disponíveis;
- NGINX Ingress e Metrics Server instalados quando necessários.

## Secrets

| Namespace | Secret | Chaves |
| --- | --- | --- |
| auth-service | auth-service-secret | DATABASE_URL, MASTER_KEY |
| flag-service | flag-service-secret | DATABASE_URL |
| targeting-service | targeting-service-secret | DATABASE_URL |
| evaluation-service | evaluation-service-secret | SERVICE_API_KEY |
| evaluation-service | aws-secret | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN |
| analytics-service | aws-secret | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN |

Não commite credenciais, tokens, URLs de banco com senha ou Secrets Kubernetes.

## Registrar as Applications

Com o kubeconfig apontando para o cluster correto:

```bash
kubectl apply -k argocd/
kubectl get applications -n argocd
kubectl get applications -n argocd -w
```

O comando registra as Applications. A partir daí, alterações na branch main
são observadas e sincronizadas automaticamente pelo Argo CD.

Além das Applications de aplicação, `otel-collector-minikube` instala o OpenTelemetry
Collector no namespace `observability` usando o chart oficial pinado. O serviço é
interno (`ClusterIP`) e recebe OTLP em:

```text
otel-collector.observability.svc.cluster.local:4317  # OTLP gRPC
otel-collector.observability.svc.cluster.local:4318  # OTLP HTTP/protobuf
```

Traces são enviados ao New Relic por OTLP/HTTP autenticado, métricas são expostas em Prometheus e logs
são enviados ao Loki em `loki-minikube.observability.svc.cluster.local:3100`. O
ServiceMonitor coleta as métricas internas do Collector. Health (`13133`) e
métricas (`8888/8889`) não são publicados por Ingress ou LoadBalancer.

O Collector usa ServiceAccount dedicado sem token montado e NetworkPolicy restrita
aos namespaces dos serviços, observabilidade, Loki e DNS. Nenhuma credencial é
versionada. A ativação de Datadog ou New Relic, incluindo o Secret externo e o
exporter OTLP correspondente, permanece como escopo do RIC-72; o ConfigMap
`otel-collector-apm-contract` documenta as chaves esperadas.

## Observabilidade Minikube

A Application `observability-minikube` instala o chart `kube-prometheus-stack` no
namespace `observability`. O Prometheus coleta métricas do cluster através do
Kubernetes API, kubelet/cAdvisor, node-exporter e kube-state-metrics. Os
seletores de `ServiceMonitor` e `PodMonitor` aceitam recursos de qualquer
namespace, permitindo que workloads instrumentados sejam adicionados sem
alterar a Application.

Depois da sincronização, valide os componentes e os targets:

```bash
kubectl get pods -n observability
kubectl port-forward -n observability svc/observability-minikube-kube-prometheus-prometheus 9090:9090
```

Em outro terminal:

```bash
curl -s http://localhost:9090/api/v1/targets \
  | jq -r '.data.activeTargets[] | [.labels.job, .health, .scrapeUrl] | @tsv'
```

Os targets de `kube-state-metrics`, `node-exporter` e kubelet devem aparecer
como `up`. Para que um serviço da aplicação apareça como target, ele precisa
expor métricas no formato Prometheus em `/metrics` e ter um `ServiceMonitor` ou
`PodMonitor`; os endpoints `/health` existentes não são endpoints de métricas.

## Validar os manifests

```bash
kubectl kustomize argocd/
kubectl kustomize services/auth-service >/dev/null
kubectl kustomize services/flag-service >/dev/null
kubectl kustomize services/targeting-service >/dev/null
kubectl kustomize services/evaluation-service >/dev/null
kubectl kustomize services/analytics-service >/dev/null
kubectl kustomize argocd/ >/dev/null
```

A renderização não valida credenciais, conectividade com a AWS ou disponibilidade
dos serviços externos.

Após sincronizar no cluster Minikube, valide o Collector com:

```bash
kubectl -n argocd get application otel-collector-minikube -o yaml
kubectl -n observability get pods,svc,servicemonitor,networkpolicy
kubectl -n observability port-forward svc/otel-collector 13133:13133
curl -fsS http://localhost:13133/
kubectl -n observability logs deploy/otel-collector --all-containers
```

O estado esperado da Application é `Synced`/`Healthy`. A indisponibilidade
temporária do Loki é absorvida pela fila e retry configurados. O exporter
`otlphttp/newrelic` também possui fila em memória e retry, sem persistência após reinício.

## Configuração dos serviços

Os Ingress expõem auth, flags, targeting e evaluation pelo NGINX. O
analytics-service é um worker e não possui API pública, mantendo apenas seu
endpoint de health. Os HPAs de evaluation e analytics dependem do Metrics Server;
os valores de CPU permanecem definidos nos manifests deste repositório.

## Relação entre os repositórios

1. fase3-terraform cria a plataforma AWS e instala EKS, Argo CD e NGINX.
2. fase3-apps constrói, testa, verifica e publica as imagens no ECR.
3. fase3-apps atualiza este repositório com a nova tag da imagem.
4. Argo CD sincroniza o Git e atualiza os workloads no EKS.

### Convenção de labels — logs Minikube

O dashboard **Microserviços — Logs Minikube** usa a fonte provisionada **Loki Minikube** (`logs-minikube`) e seletores limitados ao período consultado. `k8s_namespace_name` e `k8s_deployment_name` identificam recursos Kubernetes reais; `service_name` identifica a aplicação e `k8s_cluster_name` identifica o cluster. O Collector não injeta mais `service.namespace=togglemaster`. Pod e node permanecem atributos no JSON para evitar aumentar o número de streams. O label `namespace` das métricas Prometheus pertence à coleta e não deve ser usado como substituto do namespace de origem dos logs. Logs antigos preservam os labels originais até expirar a retenção; use um intervalo recente no dashboard novo.


## Secret do New Relic e pipeline de traces

AWS e Minikube usam o Collector para enviar traces ao New Relic por HTTPS/443.
Somente o Collector recebe a ingest license key por `secretKeyRef`:

| Namespace | Secret | Chave |
| --- | --- | --- |
| observability | new-relic-otlp | license-key |

Crie a Secret **antes** de sincronizar o Collector. O valor real não deve ser versionado.
Exemplo, usando um arquivo local protegido com somente o valor da ingest license key:

```bash
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
kubectl -n observability create secret generic new-relic-otlp \
  --from-file=license-key=/caminho/seguro/new-relic-license-key \
  --dry-run=client -o yaml | kubectl apply -f -
```

O arquivo deve conter a chave sem quebra de linha final. Também é possível provisionar
essa mesma Secret por um gerenciador externo de segredos. Ela não é um recurso gerenciado
por estas Kustomizations. Sem a Secret/chave, o pod do Collector não inicia.
Após rotação, reinicie o Deployment `otel-collector` para recarregar a variável.

As Applications `argocd/{aws,minikube}/otel-collector-*.yaml` definem o endpoint público
`https://otlp.eu01.nr-data.net`, correspondente à conta New Relic na região EU.
O exporter acrescenta `/v1/traces`.
A NetworkPolicy já permite saída TCP/443. Métricas seguem para Prometheus e logs para Loki.
Os serviços usam o endpoint interno do Collector e não precisam da chave New Relic.

Publique as novas imagens Go e atualize as tags dos Deployments pelo fluxo existente;
a configuração sozinha não adiciona spans aos binários antigos. No Minikube, reconstrua
as imagens locais e reinicie os workloads. PostgreSQL (auth), Redis e publicação SQS
(evaluation) passam a ter spans. No Minikube, SQS só terá spans se for configurado.
O consumidor Python de SQS ainda precisa extrair os MessageAttributes W3C para
continuar o trace na etapa de consumo.

Valide no New Relic após gerar tráfego, incluindo um cache miss e envio SQS em AWS:

```sql
FROM Span SELECT count(*) FACET service.name, name SINCE 15 minutes ago
```

Consulte os logs do Collector para erros de autenticação/ingestão sem imprimir a Secret.
Referência: https://docs.newrelic.com/docs/opentelemetry/best-practices/opentelemetry-otlp/

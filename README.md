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
│   └── <serviço>.yaml             # cinco Applications
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

Existem dois ambientes: `local` e `prod`.

O ambiente `local` é executado nesta máquina dentro de um cluster Minikube.
Minikube é a implementação do cluster local, não um terceiro ambiente. O
overlay público é `overlays/local`; internamente ele reutiliza os manifests
compatíveis com Minikube em `overlays/minikube`.

O ambiente local executa auth-service, flag-service, targeting-service e
evaluation-service com PostgreSQL e Redis dentro do cluster. SQS e DynamoDB não
são usados; analytics-service fica fora deste overlay.

```bash
kubectl apply -f argocd/root-local.yaml
kubectl get application togglemaster-local -n argocd
```

Para produção, continue usando a Application raiz criada pelo `fase3-terraform`
e as cinco Applications de `argocd/`. Não aplique `root-local.yaml` em um
cluster EKS.

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

O comando registra as cinco Applications. A partir daí, alterações na branch main
são observadas e sincronizadas automaticamente pelo Argo CD.

## Observabilidade local

A Application `observability-local` instala o chart `kube-prometheus-stack` no
namespace `observability`. O Prometheus coleta métricas do cluster através do
Kubernetes API, kubelet/cAdvisor, node-exporter e kube-state-metrics. Os
seletores de `ServiceMonitor` e `PodMonitor` aceitam recursos de qualquer
namespace, permitindo que workloads instrumentados sejam adicionados sem
alterar a Application.

Depois da sincronização, valide os componentes e os targets:

```bash
kubectl get pods -n observability
kubectl port-forward -n observability svc/observability-local-kube-prometheus-prometheus 9090:9090
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
```

A renderização não valida credenciais, conectividade com a AWS ou disponibilidade
dos serviços externos.

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

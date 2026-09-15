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

Não há prune automático nem self-heal configurados.

## Estrutura

`text
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
`

Os serviços são auth-service, flag-service, targeting-service,
evaluation-service e analytics-service. Os Secrets não são versionados neste
repositório.

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

`bash
kubectl apply -k argocd/
kubectl get applications -n argocd
kubectl get applications -n argocd -w
`

O comando registra as cinco Applications. A partir daí, alterações na branch main
são observadas e sincronizadas automaticamente pelo Argo CD.

## Validar os manifests

`bash
kubectl kustomize argocd/
kubectl kustomize services/auth-service >/dev/null
kubectl kustomize services/flag-service >/dev/null
kubectl kustomize services/targeting-service >/dev/null
kubectl kustomize services/evaluation-service >/dev/null
kubectl kustomize services/analytics-service >/dev/null
`

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

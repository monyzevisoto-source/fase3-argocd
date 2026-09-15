# ToggleMaster — GitOps com Argo CD

Este repositório contém os manifests Kubernetes e as Applications do Argo CD
para os cinco microsserviços do ToggleMaster: `auth-service`, `flag-service`,
`targeting-service`, `evaluation-service` e `analytics-service`.


Manifestos Kubernetes dos cinco microsserviços, mantidos neste repositório GitOps.
Cada diretório `services/<serviço>` tem uma Kustomization independente; `argocd/`
contém as cinco Applications. Não aponte uma Application para a raiz do repositório.

## Preparação

1. Use o repositório GitHub `monyzevisoto-source/fase3-argocd` como fonte GitOps.
2. As cinco Applications já apontam para a URL real e para a branch `main`; ajuste esses valores somente se o repositório ou a branch mudarem.
3. Cadastre o repositório no Argo CD se ele for privado. Os exemplos usam o projeto `default`, Argo CD no namespace `argocd` e o mesmo cluster onde ele está instalado.
4. Confirme as imagens ECR e os endpoints de Redis, SQS e DynamoDB nos Deployments e ConfigMaps. Foram preservados os valores de `fase3-apps`; sua disponibilidade não foi verificada.
5. Prepare os Secrets abaixo nos respectivos namespaces antes de sincronizar os serviços.

## Secrets externos ao Git

Não há manifestos de Secret neste repositório. Provisione-os pelo processo de gestão de segredos do ambiente. Se já existem no cluster, preserve-os.

| Namespace | Secret | Chaves obrigatórias |
| --- | --- | --- |
| auth-service | auth-service-secret | DATABASE_URL, MASTER_KEY |
| flag-service | flag-service-secret | DATABASE_URL |
| targeting-service | targeting-service-secret | DATABASE_URL |
| evaluation-service | evaluation-service-secret | SERVICE_API_KEY |
| evaluation-service | aws-secret | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN |
| analytics-service | aws-secret | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN |

Crie os namespaces usando `kubectl apply -f services/<serviço>/namespace.yaml` antes de provisionar os Secrets. A SERVICE_API_KEY deve ser uma chave válida emitida pelo auth-service. As credenciais AWS temporárias precisam ser renovadas; as referências existentes foram preservadas.

PostgreSQL e os schemas de `fase3-apps/{auth,flag,targeting}-service/db/init.sql`, Redis com TLS, SQS e DynamoDB precisam estar disponíveis. O cluster precisa conseguir baixar as imagens ECR, ter o controlador de Ingress `nginx` para as rotas e Metrics Server para os HPAs.

## Registrar e sincronizar

Depois de publicar os arquivos e configurar a URL:

```sh
kubectl apply -k argocd/
```

Esse comando registra as Applications. A sincronização é automática após aplicar as Applications: alterações na branch `main` são sincronizadas pelo Argo CD. Prepare as dependências e os Secrets antes de registrar as Applications. Não foi configurado prune automático nem self-heal.

Os HPAs mantêm os valores originais (analytics: 5% de CPU; evaluation: 70%). Os Deployments com HPA omitem `spec.replicas` para evitar disputa com o autoscaling. Os demais mantêm uma réplica.

## Imagens e atualização

As imagens usam tags imutáveis no formato `vMAJOR.MINOR.PATCH-<sha de 7 caracteres>`, publicadas pelo GitHub Actions no Amazon ECR. Após o push da imagem, o workflow do serviço altera somente o campo `image` do Deployment correspondente e cria um commit `deploy: update ...` neste repositório.

O Argo CD monitora a branch `main`, detecta a diferença entre o estado desejado no Git e o estado atual do cluster e executa o sync automático. O Kubernetes realiza então um rolling update dos pods, validando as readiness e liveness probes. Pull requests não atualizam este repositório.

## Validação local

```sh
kubectl kustomize argocd/
for service in auth-service flag-service targeting-service evaluation-service analytics-service; do
  kubectl kustomize "services/$service" >/dev/null || exit 1
done
```

A renderização local não valida conectividade, credenciais ou disponibilidade das APIs no cluster.

Referência: [Kustomize no Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/).

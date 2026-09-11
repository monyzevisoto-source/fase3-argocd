# ToggleMaster — GitOps com Argo CD

Manifestos dos cinco microserviços, derivados de `fase3-apps/*/k8s`.
Cada diretório `services/<serviço>` tem uma Kustomization independente; `argocd/`
contém as cinco Applications. Não aponte uma Application para a raiz do repositório.

## Preparação

1. Publique esta pasta em um repositório Git. Ela foi recebida vazia, sem `.git` ou remote configurado.
2. Substitua `https://github.com/SEU_USUARIO/fase3-argocd.git` nos cinco arquivos de `argocd/` pela URL real e ajuste `targetRevision` se a branch não for `main`.
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

Esse comando registra as Applications. A sincronização é manual pela interface ou CLI do Argo CD. Sincronize auth-service, depois flag-service e targeting-service, e então evaluation-service e analytics-service. Não foi configurado prune automático.

Os HPAs mantêm os valores originais (analytics: 5% de CPU; evaluation: 70%). Os Deployments com HPA omitem `spec.replicas` para evitar disputa com o autoscaling. Os demais mantêm uma réplica.

## Imagens e atualização

As imagens mantêm a tag `latest` da origem. Para cada release, prefira alterar a imagem para uma tag imutável ou digest e publicar o commit neste repositório. Enviar uma nova imagem para a mesma tag `latest` não altera o template do Deployment e não inicia um rollout por si só.

## Validação local

```sh
kubectl kustomize argocd/
for service in auth-service flag-service targeting-service evaluation-service analytics-service; do
  kubectl kustomize "services/$service" >/dev/null || exit 1
done
```

A renderização local não valida conectividade, credenciais ou disponibilidade das APIs no cluster.

Referência: [Kustomize no Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/).

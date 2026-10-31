# Réplica Kubernetes do CRM — cliente ISM

Pacote operacional para a réplica ISM no EKS existente `cluster-uze` (`us-east-1`). O ZIP foi organizado para ser extraído na raiz do repositório `uze-collection-crm`, branch `ism-prd`, e colocar estes arquivos em `.pipelines/ism-k8s-replica/`.

O pacote inclui manifests, pipeline, scripts e documentação da réplica; não inclui os fontes da aplicação. Os contextos de build existentes permanecem na raiz do repositório: `frontend/`, `backend/` e `boleto-pdf-generator/`. Não mova esses diretórios para `.pipelines/ism-k8s-replica/`.

## Conteúdo

- `00-namespace.yaml`: namespace `ism` e labels de ownership.
- `01-deployments.yaml`: frontend, Application, WebAPI, Webhooks, Workers e gerador de boletos.
- `02-services.yaml`: Services `ClusterIP` usados pela aplicação.
- `03-ingress.yaml`: hosts ISM encaminhados pelo NGINX compartilhado, com TLS via cert-manager.
- `build-and-push-ism.sh`: build/push do frontend, quatro imagens do backend ou gerador de boletos para um ECR.
- `deploy-ism.sh`: deploy local `--infra-only` ou deploy completo por Build ID; preserva os preflights de ambiente, ECR, Secret e hosts.
- `azure-pipelines-ism-prd.yml`: validação de PR, bootstrap, detecção incremental, build/push e deploy opt-in.
- `RUNBOOK-APLICACAO-ISM.md` e `.pdf`: procedimentos operacionais detalhados.

## Fluxo da pipeline

A pipeline continua limitada à branch `ism-prd`. PRs fazem apenas validação/compilação .NET, sem credenciais AWS/EKS. Em execuções não PR, o BootstrapISM valida checkout, conta/contexto, ownership e hosts; aplica namespace, Services e Ingress. A espera do namespace usa `.status.phase=Active`, e o preflight ignora somente Ingresses temporários `cm-acme-http-solver-*` no namespace `ism`; colisões com outros Ingresses continuam bloqueando.

O estágio `EnsureISMECR` mantém o ECR único `uze-crm-ism-prd`. O detector compara o commit atual com o commit da última execução bem-sucedida da mesma pipeline e branch. As regras são:

| Caminho alterado | Imagens construídas |
| --- | --- |
| `frontend/` | `front` |
| `backend/` | `application`, `webapi`, `webhooks`, `workers` |
| `boleto-pdf-generator/` | `boleto` |
| YAML da pipeline ou `build-and-push-ism.sh` | Todos os grupos |
| Manifests ISM, README/runbooks conhecidos | Nenhuma imagem |
| Arquivo não mapeado ou dependência compartilhada | Todos os grupos, por segurança |

Na primeira execução, ou se não houver histórico acessível, não houver `System.AccessToken`/permissão de leitura ou o commit anterior não estiver no checkout, o detector faz fallback para build completo. A identidade da pipeline precisa conseguir ler o histórico de builds do projeto; o detector usa checkout com `fetchDepth: 0`.

`deployWorkloads` permanece `false` por padrão. Com `false`, os grupos de build rodam apenas quando seus arquivos mudam. **Com `deployWorkloads=true`, todos os grupos são forçados a buildar**, para que as seis tags do Build ID atual existam antes do DeployISM. Uma rodada incremental parcial não deve ser usada para o deploy manual de todas as imagens.

As tags são específicas por componente e Build ID: `front-v<ID>`, `application-v<ID>`, `webapi-v<ID>`, `webhooks-v<ID>`, `workers-v<ID>` e `boleto-v<ID>`. O bootstrap/ECR continuam executando nas execuções da branch; a detecção incremental controla os estágios de build, não muda os gatilhos de branch.

## Segredo da aplicação

Durante o estágio `DeployISM`, a pipeline lê `SecretString` do AWS Secrets Manager em `us-east-1`, segredo `uze-collection-crm/ism`, e cria/atualiza o Kubernetes Secret `ism-crm-secrets` no namespace `ism`. Nenhum valor é impresso deliberadamente nos logs. O processo usa arquivos temporários protegidos e os remove ao terminar.

O formato recomendado é JSON com a connection string no campo aninhado `ConnectionStrings.Default`; esse campo é convertido em `ConnectionStrings__Default`, conforme a convenção de configuração por ambiente do .NET. As demais folhas escalares do JSON também são convertidas para nomes com `__`, para que as demais configurações da aplicação sejam preservadas. A pipeline também aceita a chave JSON já achatada `ConnectionStrings__Default` ou, se o segredo for intencionalmente apenas uma string, uma connection string em texto simples.

Exemplo **sem credenciais reais**:

```json
{
  "ConnectionStrings": {
    "Default": "Host=<endpoint>;Port=5432;Database=<database>;Username=<usuario>;Password=<senha>;SSL Mode=Require"
  }
}
```

Mantenha aspas JSON válidas e não deixe `Default` vazio. Não apague as demais configurações do segredo se a aplicação precisar delas. A pipeline interrompe o deploy se não conseguir produzir `ConnectionStrings__Default` não vazio ou se as chaves não forem nomes válidos de variáveis de ambiente.

A service connection AWS precisa de `secretsmanager:GetSecretValue` restrita ao ARN do segredo. O acesso Kubernetes precisa permitir validar/criar/atualizar Secret no namespace `ism` e aplicar os manifests do deploy. Restrinja essas permissões; confirme também a criptografia de Secrets em repouso e o RBAC de leitura. Rotacione qualquer credencial que tenha sido exposta em imagem, log ou chat. O deploy não prova que usuário, senha, TLS, grants ou conectividade com RDS estão corretos.

O script local `deploy-ism.sh` **não** sincroniza o Secrets Manager: ele verifica apenas a presença da chave no Secret Kubernetes existente. A sincronização acima ocorre no estágio `DeployISM` da pipeline.

## Preparação no Azure DevOps

1. Extraia o ZIP na raiz do clone limpo da branch `ism-prd`; o diretório resultante deve ser `.pipelines/ism-k8s-replica/`.
2. Confirme que o YAML da definição aponta para `/.pipelines/ism-k8s-replica/azure-pipelines-ism-prd.yml`.
3. Autorize o Variable Group `uze-collection-crm-ISM-PRD` e a service connection AWS. O grupo fornece `awsCredentials`, `awsRegion`, `awsAccountId`, `eksClusterName`, `ApiUrl` e `ManusApi__BaseUrl`; não coloque nele connection strings, senhas ou tokens.
4. Garanta permissões ECR para criar/descrever o repositório e publicar imagens, acesso autorizado ao cluster EKS/Kubernetes, leitura de builds para o detector e `GetSecretValue` no segredo ISM. Para aplicação de Secrets, o RBAC precisa permitir criação/atualização restrita ao namespace `ism`.
5. Faça commit/push do pacote na branch `ism-prd`. Inspecione o resumo de `DetectChanges` e quais estágios de build foram executados.

## Execução e conferências

Faça primeiro uma execução com `deployWorkloads=false`. Aguarde bootstrap e build; na primeira execução, o fallback pode construir todas as imagens por não haver execução anterior bem-sucedida. Confirme Build ID e tags no ECR. Só habilite `deployWorkloads=true` após provisionar database/usuário ISM, validar conectividade PostgreSQL a partir de uma origem de rede apropriada, conferir o segredo no AWS Secrets Manager e aprovar a janela de deploy. A rodada com deploy faz build completo e sincroniza o Secret antes de aplicar workloads.

Comandos read-only úteis:

```bash
kubectl.exe get namespace ism -o jsonpath='{.status.phase}{"\n"}'
kubectl.exe get services,ingress -n ism -o wide
kubectl.exe get deployments,pods -n ism -o wide
kubectl.exe get endpointslices -n ism
kubectl.exe get events -n ism --sort-by=.lastTimestamp | tail -40
```

Verifique a presença da chave do Kubernetes Secret sem mostrar seu valor:

```bash
kubectl.exe get secret ism-crm-secrets -n ism \
  -o go-template='{{if index .data "ConnectionStrings__Default"}}presente{{else}}ausente{{end}}{{"\n"}}'
```

O bootstrap cria Services e Ingress mesmo sem Pods; Services sem endpoints ou HTTP 503 podem ser esperados até o deploy. Verifique hosts/DNS/TLS e health endpoints próprios da aplicação. A raiz de API/webhooks pode retornar status diferente de 2xx; não desabilite validação TLS como workaround.

## Uso local e rollback

Use `deploy-ism.sh` a partir de `.pipelines/ism-k8s-replica/`, com AWS CLI, `jq`, `sed` e `kubectl.exe` configurados para o cluster esperado:

```bash
./deploy-ism.sh --infra-only
read -rp 'Build ID completo: ' BUILD_ID
[[ "$BUILD_ID" =~ ^[0-9]+$ ]] || { echo 'Build ID inválido' >&2; exit 1; }
./deploy-ism.sh "$BUILD_ID"
```

O modo `--infra-only` aplica apenas namespace, Services e Ingress. O modo completo exige as seis tags do Build ID e um Secret `ism-crm-secrets` já existente com a chave esperada. Ele valida conflitos de host, incluindo wildcard, ignora somente os solver Ingresses temporários do cert-manager no namespace ISM e espera pelo phase `Active` do namespace. Não apaga nem recria namespace como workaround.

Para rollback, use um Build ID anterior saudável e confira as seis tags antes de mudar as imagens. Rollback de imagem não desfaz migrations, alterações no database, filas ou integrações. Não apague o namespace, Secret, database ou ECR como rollback normal.

## Limites desta entrega

O ZIP contém o conteúdo completo da pasta `.pipelines/ism-k8s-replica/`, com documentação atualizada; não contém os fontes `frontend/`, `backend/` ou `boleto-pdf-generator/`, que permanecem no repositório existente. Nenhuma pipeline, recurso AWS ou objeto Kubernetes foi executado/modificado durante a preparação deste pacote.

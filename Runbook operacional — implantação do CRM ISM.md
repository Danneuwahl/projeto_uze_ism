# Runbook operacional — implantação do CRM ISM

**Ambiente de referência:** EKS `cluster-uze` · região `us-east-1` · conta `189065795615`  
**Namespace:** `ism` · **Branch:** `ism-prd` · **ECR:** `uze-crm-ism-prd`  
**Hosts:** `ism-crm.collectioncenter.com.br`, `ism-api-crm.collectioncenter.com.br`, `ism-webhooks-crm.collectioncenter.com.br`  
**Versão:** 1.1 · **Atualizado:** 31/10/2026

> Confirme os valores no Variable Group e no Azure DevOps antes de executar. Este documento descreve a versão fornecida para revisão; não significa que a pipeline ou o deploy tenham sido executados.

## 1. Objetivo, escopo e proteção de segredos

Este procedimento cobre a pasta `.pipelines/ism-k8s-replica/`, o bootstrap de infraestrutura no cluster existente, o build/push seletivo para ECR e a implantação controlada dos workloads ISM. Não cria database/usuário RDS, DNS, certificados, Load Balancer, node group nem recursos para `prd-uze`/`hml-uze`.

A pipeline só é acionada para a branch dedicada `ism-prd`. O bootstrap prepara namespace, Services e Ingress. O parâmetro `deployWorkloads` continua `false` por padrão; habilite-o somente após preparar e testar banco, segredo, rede, imagens e janela de mudança.

**Nunca** cole connection strings, senhas, chaves ou tokens em Git, Variable Group, logs, comandos, tickets ou chat. Não use `kubectl get secret -o yaml/json`, `printenv` do Pod ou consultas que decodifiquem `data`. Se um segredo for exibido em captura/log, trate-o como comprometido, rotacione-o e atualize o cofre antes do deploy.

## 2. Pacote e caminhos

O ZIP atualizado está pronto para ser extraído na raiz do clone `uze-collection-crm`. Ele contém:

```text
.pipelines/ism-k8s-replica/
  00-namespace.yaml
  01-deployments.yaml
  02-services.yaml
  03-ingress.yaml
  azure-pipelines-ism-prd.yml
  build-and-push-ism.sh
  deploy-ism.sh
  README.md
  RUNBOOK-APLICACAO-ISM.md
  RUNBOOK-APLICACAO-ISM.pdf
```

Os contextos Docker permanecem na raiz do repositório (`frontend/`, `backend/`, `boleto-pdf-generator/`). O ZIP cobre os artefatos operacionais ISM; não contém nem move o código-fonte da aplicação.

Confirme os caminhos no checkout real antes do commit. A definição Azure Pipelines deve apontar para `/.pipelines/ism-k8s-replica/azure-pipelines-ism-prd.yml`.

## 3. Fluxo e tags de imagem

### Pipeline normal (`deployWorkloads=false`)

1. **ValidatePR:** PR para `ism-prd` executa compilação/validação .NET, sem credenciais AWS/EKS.
2. **BootstrapISM:** confere arquivos, conta/contexto, ownership do namespace/Ingress e colisão de hosts; aplica namespace, Services e Ingress.
3. **EnsureISMECR:** garante o repositório único `uze-crm-ism-prd` (scan-on-push e criptografia AES256).
4. **DetectChanges:** compara o commit atual com o commit da execução concluída com sucesso mais recente da mesma definição/branch.
5. **BuildFrontend / BuildBackend / BuildBoletoPdfGenerator:** somente os grupos marcados pelo detector executam build/push.
6. **DeployISM:** permanece desabilitado porque `deployWorkloads=false`.

O bootstrap e a validação do ECR continuam ocorrendo nas execuções da branch. A detecção controla builds de imagem; ela não muda gatilhos nem desativa o bootstrap.

### Regras da detecção incremental

| Caminho do diff | Grupo de build |
| --- | --- |
| `frontend/` | frontend → tag `front` |
| `backend/` | Application, WebAPI, Webhooks e Workers |
| `boleto-pdf-generator/` | gerador → tag `boleto` |
| `.pipelines/ism-k8s-replica/build-and-push-ism.sh` ou YAML da pipeline | todos os grupos |
| Manifests ISM e documentação explicitamente listados | nenhum grupo de imagem |
| Caminho não mapeado/dependência compartilhada | todos os grupos, como fallback conservador |

O detector consulta o histórico de builds do projeto com `System.AccessToken`, usa checkout com `fetchDepth: 0` e calcula `git diff --name-only` entre `sourceVersion` anterior e atual. A identidade de build precisa de leitura mínima de builds (`View builds`/Build Read) no projeto. Garanta que o token OAuth do job esteja disponível para o script; o pipeline já o mapeia ao ambiente e não imprime seu conteúdo.

Na primeira execução, se não existir execução anterior bem-sucedida, faltar token/permissão, a consulta falhar ou o commit base não estiver disponível, o detector faz **build completo**. Caminhos não mapeados também fazem build completo. Esse fallback pode custar mais, mas evita ignorar dependências compartilhadas.

### Execução de deploy (`deployWorkloads=true`)

O parâmetro de deploy força os três grupos de build, mesmo que o diff seja pequeno, para que todas as seis tags do Build ID corrente estejam no ECR. O DeployISM valida as imagens, lê/sincroniza o Secret do Secrets Manager, faz server-side dry-run e só então aplica o Secret e os Deployments no namespace `ism`.

Tags por Build ID: `front-v<ID>`, `application-v<ID>`, `webapi-v<ID>`, `webhooks-v<ID>`, `workers-v<ID>` e `boleto-v<ID>`. Uma execução incremental parcial não é um pacote completo para um deploy manual de todos os serviços. Para deploy de workloads, habilite o parâmetro e deixe a pipeline gerar todas as tags daquela execução.

## 4. Bootstrap e proteção do Ingress

O namespace aguarda por `.status.phase=Active` usando `kubectl wait --for=jsonpath='{.status.phase}'=Active`; não usar `--for=condition=Active`, pois `Active` não é uma condition do Namespace.

O preflight global confere os três hosts e wildcards. Ele ignora somente `ism/ism-crm-ingress` (Ingress gerenciado do próprio pacote) e Ingresses temporários `cm-acme-http-solver-*` do cert-manager no namespace `ism`, que precisam usar os hosts durante o desafio HTTP-01. Qualquer outro Ingress sobre os hosts continua sendo conflito e bloqueia o deploy; não apague nem altere recurso de outro tenant para contornar a checagem.

Os Services permanecem `ClusterIP`. Não introduzir `NodePort`, `hostPort` ou `hostNetwork` como solução de tráfego. O Ingress usa a classe NGINX existente e o `ClusterIssuer` configurado no cluster; DNS e TLS precisam de validação separada.

## 5. Pré-flight de operador

Execute no Git Bash com AWS CLI, `kubectl.exe`, `jq`, Docker e Git disponíveis. Use apenas o perfil/contexto autorizado:

```bash
aws --version
kubectl.exe version --client
jq --version
git --version
aws sts get-caller-identity --profile uze
aws eks describe-cluster --profile uze --region us-east-1 --name cluster-uze \
  --query 'cluster.{arn:arn,status:status}' --output yaml
kubectl.exe config current-context
kubectl.exe get nodes -l eks.amazonaws.com/nodegroup=workers \
  -L eks.amazonaws.com/nodegroup,hml-uze,topology.kubernetes.io/zone -o wide
kubectl.exe top nodes
```

Esperado: conta `189065795615`, cluster `cluster-uze` `ACTIVE` e contexto EKS correspondente. Confirme capacidade e que existem nós elegíveis: os Deployments ISM excluem nós com label `hml-uze`, mas isso não garante isolamento dedicado de ISM em relação a PRD.

Verifique classe e issuer compartilhados:

```bash
kubectl.exe get ingressclass nginx -o yaml
kubectl.exe get clusterissuer letsencrypt-prod
kubectl.exe get ingress -A \
  -o 'custom-columns=NS:.metadata.namespace,NAME:.metadata.name,HOSTS:.spec.rules[*].host'
```

Pare se conta/contexto estiverem errados, houver `Forbidden`, ownership desconhecido, conflito de host ou falta de capacidade. Não recrie namespace, controller, issuer ou Load Balancer como workaround.

## 6. Preparação do Azure DevOps e permissões

1. Extraia o ZIP atualizado na raiz do clone limpo da branch `ism-prd`; confira que os arquivos ficaram em `.pipelines/ism-k8s-replica/`.
2. Confirme a definição YAML e a referência acessível `UZE - CRM/devops-templates` (`main`).
3. Autorize o Variable Group `uze-collection-crm-ISM-PRD` com as variáveis atuais: `awsCredentials`, `awsRegion=us-east-1`, `awsAccountId=189065795615`, `eksClusterName=cluster-uze`, `ApiUrl=https://ism-api-crm.collectioncenter.com.br` e `ManusApi__BaseUrl` validado. Não guarde connection strings ou tokens nele.
4. Conceda à service connection somente as permissões necessárias para ECR, `eks:DescribeCluster` e autenticação Kubernetes. O mecanismo de acesso do EKS/RBAC deve autorizar os comandos previstos e limitar alteração namespaced ao namespace ISM sempre que possível. Criar Namespace/consultar IngressClass/ClusterIssuer e listar Ingresses requerem leitura fora do namespace; conceda-a de forma explícita e mínima.
5. Para o detector, conceda leitura do histórico de builds à identidade do pipeline. Se a API não estiver acessível, o fallback de segurança será build completo.
6. Para a sincronização do segredo, conceda `secretsmanager:GetSecretValue` restrita ao ARN de `uze-collection-crm/ism`. A identidade Kubernetes usada no DeployISM precisa de `get`, `create`, `patch`/`update` para Secret em `ism`, além dos verbos necessários para aplicar os Deployments e recursos relacionados.
7. Confirme que a role dos nós pode puxar imagens do ECR. A role que publica imagens e a role do node group são distintas.

O ECR usa um único repositório `uze-crm-ism-prd`, scan-on-push e criptografia AES256. As tags são MUTABLE, mas cada push normal usa nome do componente + Build ID; avalie retenção/lifecycle separadamente após definir o período de rollback.

## 7. Primeira execução e leitura dos resultados

Faça commit/push dos arquivos na branch `ism-prd`. Na primeira execução, mantenha `deployWorkloads=false`. Como ainda pode não existir execução anterior bem-sucedida para comparação, o detector deve optar por build completo. Nos logs, confirme:

- branch/conta/contexto esperados e bootstrap concluído;
- resumo de `DetectChanges` e motivo de build completo, se aplicável;
- execução dos grupos de build esperados;
- seis tags com o mesmo Build ID quando o detector fizer build completo;
- ausência de DeployISM com o parâmetro `false`.

Confira as tags de um Build ID concluído:

```bash
read -rp 'Build ID concluído: ' BUILD_ID
[[ "$BUILD_ID" =~ ^[0-9]+$ ]] || { echo 'Build ID inválido' >&2; exit 1; }
for component in front application webapi webhooks workers boleto; do
  aws ecr describe-images --profile uze --region us-east-1 \
    --repository-name uze-crm-ism-prd \
    --image-ids imageTag="${component}-v${BUILD_ID}" \
    --query 'imageDetails[0].{tag:imageTags[0],digest:imageDigest}' --output table || exit 1
done
```

Quando somente um componente mudar, deve rodar o grupo correspondente. Modificação no build script, pipeline ou arquivo compartilhado não mapeado deve rodar tudo. Mudança somente em manifests/documentação listados não precisa de imagem; BootstrapISM ainda pode rodar para aplicar a infraestrutura declarada.

## 8. Database, conectividade e AWS Secrets Manager

Crie/valide database e usuário exclusivos do ISM no RDS. Não reutilize `postgres` ou credenciais de outros ambientes para a aplicação. Confirme grants mínimos, database, porta 5432, TLS e compatibilidade de schema/migrations. A pipeline não cria o database nem comprova autenticação.

Teste DNS/porta a partir de origem autorizada na VPC e com política de rede equivalente à do Pod:

```bash
nslookup <endpoint-rds>
nc -vz -w 5 <endpoint-rds> 5432
```

Para autenticar com `psql`, use `-W` para solicitação interativa; não coloque senha em argumentos ou variável de ambiente:

```bash
psql "host=<endpoint-rds> port=5432 dbname=<database_ism> user=<usuario_ism> sslmode=require connect_timeout=5" \
  -W -c 'select current_database(), current_user;'
```

Se as ferramentas não existirem no container de aplicação, não as instale nele. Um Pod temporário com imagem `postgres:16-alpine` só deve ser usado se aprovado pela política do cluster; execute `pg_isready`/`psql` nele e remova-o após o teste. Esse teste valida o caminho de rede do namespace onde o Pod foi criado, não outro namespace com políticas diferentes.

### Formato do segredo

A pipeline consome o segredo AWS `uze-collection-crm/ism` (região `us-east-1`) durante DeployISM. Recomenda-se manter a configuração como JSON no AWS Secrets Manager, com a connection string em `ConnectionStrings.Default` e demais configurações necessárias no mesmo objeto. Exemplo sem valores reais:

```json
{
  "ConnectionStrings": {
    "Default": "Host=<endpoint>;Port=5432;Database=<database>;Username=<usuario_ism>;Password=<senha>;SSL Mode=Require"
  },
  "Security": {
    "JwtSecret": "<segredo-da-aplicacao>"
  }
}
```

No editor JSON da console AWS, `Default` precisa estar preenchido com uma string JSON válida entre aspas. Preservar as demais chaves usadas pela aplicação; não apagar o objeto inteiro para inserir apenas a conexão. A pipeline também aceita `ConnectionStrings__Default` já achatado ou `SecretString` em texto simples contendo somente a connection string. Ela exige connection string não vazia e converte seções/folhas para nomes com `__` do ambiente .NET.

O estágio obtém `SecretString`, cria manifesto temporário protegido, executa dry-run do Secret e aplica `ism-crm-secrets` antes dos Deployments. Não imprime o conteúdo. Se a leitura, o formato ou a permissão falhar, o stage deve parar antes de aplicar workloads. O script local `deploy-ism.sh` não busca AWS Secrets Manager nem cria Secrets; requer que o Secret Kubernetes já exista.

Verifique somente a presença do Secret/chave, sem mostrar o valor:

```bash
kubectl.exe get secret ism-crm-secrets -n ism \
  -o go-template='{{if index .data "ConnectionStrings__Default"}}chave presente{{else}}chave ausente{{end}}{{"\n"}}'
```

## 9. Deploy controlado

Antes de habilitar `deployWorkloads=true`, confirme database/usuário, conectividade/TLS, JSON do segredo, permissões IAM/RBAC, hosts/DNS/TLS e capacidade dos nós. Depois, execute a pipeline em `ism-prd` com o parâmetro explicitamente habilitado.

A execução força build completo dos três grupos, verifica as seis tags do Build ID atual, sincroniza o Secret do AWS Secrets Manager, renderiza o Build ID nos manifests e faz server-side dry-run antes do apply. Só então aplica namespace/Services/Secret/Deployments/Ingress no namespace `ism`. Não execute stages de deploy da pipeline PRD nem aponte um script a outro namespace.

## 10. Validação pós-deploy

```bash
for deployment in \
  ism-crm-front ism-crm-application ism-crm-webapi \
  ism-crm-webhooks ism-crm-workers ism-crm-boleto-pdf-generator; do
  kubectl.exe rollout status "deployment/${deployment}" -n ism --timeout=300s || exit 1
done
kubectl.exe get deployments,pods,services,ingress -n ism -o wide
kubectl.exe get endpointslices -n ism
kubectl.exe get events -n ism --sort-by=.lastTimestamp | tail -40
kubectl.exe top pods -n ism --containers
kubectl.exe top nodes
```

Confirme as tags sem ler ambiente/Secrets:

```bash
kubectl.exe get pods -n ism \
  -o jsonpath="{range .items[*]}{.metadata.name}{' '}{range .spec.containers[*]}{.image}{' '}{end}{'\n'}{end}"
```

Valide DNS, TLS e rotas pelos health paths documentados. Não desabilite validação TLS; status HTTP não-2xx na raiz de API/webhook pode ser esperado, mas timeout, NXDOMAIN, certificado inválido ou backend errado exigem correção.

```bash
for host in \
  ism-crm.collectioncenter.com.br \
  ism-api-crm.collectioncenter.com.br \
  ism-webhooks-crm.collectioncenter.com.br; do
  echo "=== $host ==="
  nslookup "$host"
  curl --connect-timeout 5 --max-time 15 -sS -o /dev/null \
    -w 'HTTP %{http_code} | TLS verify %{ssl_verify_result}\n' "https://${host}/"
done
kubectl.exe get certificates,certificaterequests,orders,challenges -n ism
kubectl.exe get clusterissuer letsencrypt-prod
```

Os manifests atuais não definem readiness/liveness probes. Rollout concluído significa que o Kubernetes concluiu a estratégia declarada, não que regras de negócio ou conectividade do banco foram validadas. Use smoke tests funcionais antes de liberar tráfego.

## 11. Troubleshooting

- **Detector faz build completo:** esperado no primeiro run, após mudança em pipeline/build helper/arquivo compartilhado desconhecido ou quando falha o acesso ao histórico. Se recorrente, confira `System.AccessToken`, permissão Build Read, `fetchDepth: 0` e histórico da branch.
- **Componente inesperado não builda:** verifique caminho versionado (`frontend/`, `backend/`, `boleto-pdf-generator/`) e variáveis de saída `frontendChanged`, `backendChanged`, `boletoChanged`. Caminho não mapeado deve cair para build completo.
- **Build stage foi skipped:** se o detector concluiu que não houve mudança nesse grupo e `deployWorkloads=false`, isso é esperado. Com `deployWorkloads=true`, os três grupos devem rodar.
- **Tag ausente/ImagePullBackOff:** valide Build ID, componente, publicação ECR e role de pull do node group. DeployISM requer seis tags do run atual.
- **`AccessDenied` em Secrets Manager:** conceda `secretsmanager:GetSecretValue` apenas ao ARN do segredo; não registre o SecretString para diagnosticar.
- **Secret ausente/connection string vazia/JSON inválido:** confira somente a estrutura e nomes de chaves no console; `ConnectionStrings.Default` deve ser uma string JSON válida e não vazia. Nunca copie o valor para o log.
- **Secret apply `Forbidden`:** revise RBAC/EKS access para criar/atualizar Secret no namespace `ism` e o server-side dry-run.
- **Timeout PostgreSQL:** DNS/porta/rede antes de autenticação; conferir rota, SG, NACL, NetworkPolicy e origem do teste. Erro de senha só é diagnosticável após TCP conectar.
- **Solver Ingress reportado como conflito:** solver temporário `cm-acme-http-solver-*` em `ism` deve ser ignorado pela versão atual; outros Ingresses para os hosts continuam bloqueando. Confirme que o YAML atualizado foi commitado.
- **Namespace wait timeout:** verificar `kubectl.exe get namespace ism -o jsonpath='{.status.phase}{"\n"}'`; aguardar por `jsonpath` de `status.phase`, não `condition=Active`.
- **`Pending`:** inspecionar eventos, requests/limits, label `hml-uze`, capacidade e zonas. Não remova regras de afinidade sem revisão.
- **CrashLoopBackOff:** ler logs limitados e eventos sem imprimir configurações de ambiente ou dados do Secret.

## 12. Uso do script local

O script local foi corrigido para aguardar o phase `Active` e ignorar apenas solver Ingresses temporários do cert-manager dentro de `ism`. Ele usa manifests relativos ao diretório atual; execute a partir da pasta de manifests:

```bash
cd .pipelines/ism-k8s-replica
./deploy-ism.sh --infra-only
read -rp 'Build ID completo: ' BUILD_ID
[[ "$BUILD_ID" =~ ^[0-9]+$ ]] || { echo 'Build ID inválido' >&2; exit 1; }
./deploy-ism.sh "$BUILD_ID"
```

`--infra-only` aplica namespace, Services e Ingress sem Pods. O modo completo requer que as seis tags de imagem do Build ID existam e que o Secret Kubernetes `ism-crm-secrets` já esteja provisionado. Não sincroniza o segredo da AWS. O script verifica AWS/EKS, node group/capacidade, IngressClass, ownership, tags, Secret e conflitos antes de aplicar. Leia o script e confirme perfil, contexto e branch antes de executá-lo.

## 13. Rollback

Use um Build ID anterior conhecido como saudável e confirme a existência das seis tags antes de mudar qualquer Deployment. O rollback de imagens não desfaz migrations, schema, filas, dados ou efeitos de integração externa.

```bash
read -rp 'Build ID anterior aprovado: ' PREVIOUS_BUILD_ID
[[ "$PREVIOUS_BUILD_ID" =~ ^[0-9]+$ ]] || { echo 'Build ID inválido' >&2; exit 1; }
for component in front application webapi webhooks workers boleto; do
  aws ecr describe-images --profile uze --region us-east-1 \
    --repository-name uze-crm-ism-prd \
    --image-ids imageTag="${component}-v${PREVIOUS_BUILD_ID}" \
    --query 'imageDetails[0].imageDigest' --output text || exit 1
done
```

Só depois de validar todas as tags, aplique-as aos Deployments com a equipe responsável e acompanhe cada rollout. Não apague namespace, Secrets, database ou ECR como rollback normal. `kubectl rollout undo` não substitui rollback de migrations.

## 14. Encerramento

Registre Build ID, grupos buildados, resultado de cada rollout, tags/digests, smoke tests, conectividade PostgreSQL, DNS/TLS e pendências. Se um erro permanecer, mantenha `deployWorkloads=false` nas próximas execuções até a causa e a retomada serem aprovadas.

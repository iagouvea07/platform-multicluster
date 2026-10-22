# LaboratÃ³rio Hub-Spoke com Kind, OCM, Argo CD e governanÃ§a

> **Objetivo:** montar uma prova de conceito incremental em que um cluster **Hub** gerencia o registro e a governanÃ§a dos clusters **Spoke**, enquanto cada Spoke executa seu prÃ³prio Argo CD para reconciliar suas aplicaÃ§Ãµes locais.
>
> **Escopo:** laboratÃ³rio local com `kind`. NÃ£o Ã© uma configuraÃ§Ã£o de produÃ§Ã£o. Os comandos e manifests devem ser validados com as versÃµes escolhidas antes de serem promovidos para ambientes reais.

## 1. Arquitetura e decisÃµes

```text
                         Git (repositÃ³rio)
                               |
                         Argo CD do Hub
                               |
                  configuraÃ§Ãµes locais do Hub
                  e manifests de governanÃ§a
                               |
                      OCM / clusteradm
                    /                    \
             Spoke dev                  Spoke prod
             - Klusterlet               - Klusterlet
             - OLM (opcional)           - OLM (opcional)
             - Argo CD local            - Argo CD local
             - apps de dev              - apps de prod

DistribuiÃ§Ã£o inicial pelo OCM:
Hub Argo CD -> ManifestWork no Hub -> OCM Work Agent -> recursos nos Spokes
```

### Responsabilidade de cada componente

- **Kind:** cria os clusters Kubernetes locais para o laboratÃ³rio.
- **OCM (`clusteradm`, Cluster Manager, Klusterlet):** registra e acompanha os Spokes; distribui recursos declarados em `ManifestWork`.
- **Argo CD do Hub:** gerencia recursos do prÃ³prio Hub e, depois do registro dos Spokes, pode sincronizar `ManifestWork` declarados no Git.
- **Argo CD de cada Spoke:** gerencia as aplicaÃ§Ãµes locais daquele Spoke. NÃ£o deve competir com OCM pelo controle dos mesmos objetos.
- **OLM:** opcional. Serve para instalar e administrar Operators que publiquem bundles/catalogs compatÃ­veis. **NÃ£o Ã© requisito para instalar Argo CD** e nÃ£o se deve presumir que exista um Operator oficial do Argo CD disponÃ­vel no catÃ¡logo escolhido.
- **GovernanÃ§a OCM:** pode ser adicionada posteriormente com o Governance Policy Framework/add-on, ou por polÃ­ticas e recursos distribuÃ­dos via `ManifestWork`, conforme a necessidade.

### Regra de propriedade dos recursos

| Tipo de recurso | ResponsÃ¡vel recomendado |
|---|---|
| Componentes do OCM no Hub | OCM/clusteradm |
| Argo CD do Hub e configuraÃ§Ãµes locais do Hub | Argo CD do Hub, depois do bootstrap |
| Klusterlet e recursos de registro no Spoke | OCM/clusteradm |
| InstalaÃ§Ã£o inicial do Argo CD em cada Spoke | Inicialmente operador do laboratÃ³rio; depois pode ser automatizada pelo Hub via ManifestWork |
| AplicaÃ§Ãµes de negÃ³cio no Spoke | Argo CD local do Spoke |
| PolÃ­ticas centrais de governanÃ§a | Argo CD do Hub e/ou add-on de governanÃ§a do OCM |
| Mesmo Deployment/Service | **Nunca** deixar Argo CD local e ManifestWork concorrerem para gerenciar o mesmo objeto |

## 2. PrÃ©-requisitos

Instale e valide:

- Docker
- `kind`
- `kubectl`
- `clusteradm`
- Git
- `curl`

Confira as versÃµes:

```bash
docker version
kind version
kubectl version --client
clusteradm version
git --version
```

Instale `clusteradm` usando o instalador oficial (revise o script antes de executÃ¡-lo em ambientes controlados):

```bash
curl -L https://raw.githubusercontent.com/open-cluster-management-io/clusteradm/main/install.sh | bash
```

**RecomendaÃ§Ã£o:** para um laboratÃ³rio reproduzÃ­vel, registre as versÃµes de Kind, Kubernetes, OCM/clusteradm e Argo CD usadas. Depois substitua referÃªncias flutuantes (`main`, `stable`, `latest`) por versÃµes fixadas.

## 3. Criar os trÃªs clusters Kind

Crie o Hub e dois Spokes. Use nomes explÃ­citos para simplificar o contexto do `kubectl`.

```bash
kind create cluster --name hub
kind create cluster --name spoke-dev
kind create cluster --name spoke-prod
```

Confira os contextos e nÃ³s:

```bash
kubectl config get-contexts
kubectl get nodes --context kind-hub
kubectl get nodes --context kind-spoke-dev
kubectl get nodes --context kind-spoke-prod
```

Defina variÃ¡veis para evitar aplicar recursos no cluster errado:

```bash
export CTX_HUB_CLUSTER=kind-hub
export CTX_SPOKE_DEV=kind-spoke-dev
export CTX_SPOKE_PROD=kind-spoke-prod
```

Valide cada contexto antes de continuar:

```bash
kubectl cluster-info --context "$CTX_HUB_CLUSTER"
kubectl cluster-info --context "$CTX_SPOKE_DEV"
kubectl cluster-info --context "$CTX_SPOKE_PROD"
```

> **Importante sobre Kind:** os clusters precisam conseguir alcanÃ§ar o endpoint da API do Hub. Para clusters locais, siga o procedimento do OCM para Kind e use `--force-internal-endpoint-lookup` quando indicado. Se o join falhar por conectividade ou endpoint, resolva isso antes de continuar; nÃ£o prossiga supondo que o registro funcionou.

## 4. Inicializar o OCM no Hub

Inicialize o control plane do OCM no Hub:

```bash
clusteradm init --wait --context "$CTX_HUB_CLUSTER"
```

O comando apresenta informaÃ§Ãµes e um comando/token para registrar clusters. Guarde o token com seguranÃ§a e nÃ£o o publique em Git ou logs compartilhados.

Confira os componentes:

```bash
kubectl get pods -n open-cluster-management --context "$CTX_HUB_CLUSTER"
kubectl get crd --context "$CTX_HUB_CLUSTER" | grep -E 'managedclusters|manifestworks|placements'
```

Se os componentes nÃ£o estiverem prontos, pare aqui e investigue os logs antes de registrar os Spokes.

## 5. Registrar os Spokes no OCM

Obtenha um token de join. A saÃ­da exata pode variar conforme a versÃ£o:

```bash
clusteradm get token --context "$CTX_HUB_CLUSTER"
```

Copie o token retornado para uma variÃ¡vel de ambiente no terminal. NÃ£o o coloque em arquivos versionados:

```bash
read -rsp "Token OCM do Hub: " HUB_TOKEN
echo
```

Descubra o endpoint da API do Hub que os Spokes conseguem alcanÃ§ar. Em Kind, **nÃ£o assuma que `127.0.0.1` funciona a partir de outro container**. O exemplo abaixo Ã© um placeholder: substitua-o pelo endpoint acessÃ­vel no seu ambiente.

```bash
export HUB_API_SERVER="https://ENDPOINT-ACESSIVEL-DO-HUB:PORTA"
```

Registre o Spoke de desenvolvimento:

```bash
clusteradm join \
  --hub-token "$HUB_TOKEN" \
  --hub-apiserver "$HUB_API_SERVER" \
  --wait \
  --cluster-name spoke-dev \
  --force-internal-endpoint-lookup \
  --context "$CTX_SPOKE_DEV"
```

Registre o Spoke de produÃ§Ã£o:

```bash
clusteradm join \
  --hub-token "$HUB_TOKEN" \
  --hub-apiserver "$HUB_API_SERVER" \
  --wait \
  --cluster-name spoke-prod \
  --force-internal-endpoint-lookup \
  --context "$CTX_SPOKE_PROD"
```

Aprove os pedidos no Hub:

```bash
clusteradm accept --clusters spoke-dev,spoke-prod --context "$CTX_HUB_CLUSTER"
```

Verifique:

```bash
kubectl get managedclusters --context "$CTX_HUB_CLUSTER"
kubectl get managedcluster spoke-dev --context "$CTX_HUB_CLUSTER" -o yaml
kubectl get managedcluster spoke-prod --context "$CTX_HUB_CLUSTER" -o yaml
kubectl get pods -A --context "$CTX_SPOKE_DEV" | grep open-cluster-management
kubectl get pods -A --context "$CTX_SPOKE_PROD" | grep open-cluster-management
```

**CritÃ©rio de aceite:** ambos os `ManagedCluster` devem indicar que foram aceitos e estÃ£o disponÃ­veis. Se nÃ£o estiverem `AVAILABLE=True`, nÃ£o avance.

Limpe a variÃ¡vel de token quando nÃ£o for mais necessÃ¡ria:

```bash
unset HUB_TOKEN
```

## 6. Primeiro teste ponta a ponta: ManifestWork com ConfigMap

Antes de instalar Argo CD ou OLM remotamente, prove que o OCM consegue distribuir um recurso simples.

Crie `manifests/01-test-configmap-work.yaml`:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: bootstrap-test-configmap
  namespace: spoke-dev
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Namespace
        metadata:
          name: ocm-smoke-test
      - apiVersion: v1
        kind: ConfigMap
        metadata:
          name: ocm-smoke-test
          namespace: ocm-smoke-test
        data:
          source: "OCM ManifestWork"
          cluster: "spoke-dev"
```

Aplique no Hub:

```bash
kubectl apply -f manifests/01-test-configmap-work.yaml --context "$CTX_HUB_CLUSTER"
```

Confira o status do trabalho:

```bash
kubectl get manifestwork -n spoke-dev --context "$CTX_HUB_CLUSTER"
kubectl describe manifestwork bootstrap-test-configmap -n spoke-dev --context "$CTX_HUB_CLUSTER"
```

Confira o recurso no Spoke:

```bash
kubectl get namespace ocm-smoke-test --context "$CTX_SPOKE_DEV"
kubectl get configmap ocm-smoke-test -n ocm-smoke-test --context "$CTX_SPOKE_DEV" -o yaml
```

**CritÃ©rio de aceite:** o ConfigMap aparece no Spoke e o ManifestWork informa que o recurso foi aplicado.

### Testar atualizaÃ§Ã£o e remoÃ§Ã£o

Edite `data.source` para `OCM atualizado`, reaplique e confira o valor no Spoke:

```bash
kubectl apply -f manifests/01-test-configmap-work.yaml --context "$CTX_HUB_CLUSTER"
kubectl get configmap ocm-smoke-test -n ocm-smoke-test --context "$CTX_SPOKE_DEV" -o jsonpath='{.data.source}{"\n"}'
```

Depois remova o ManifestWork:

```bash
kubectl delete -f manifests/01-test-configmap-work.yaml --context "$CTX_HUB_CLUSTER"
```

A remoÃ§Ã£o do ManifestWork normalmente remove os recursos gerenciados por ele. Verifique o comportamento de limpeza da versÃ£o usada antes de aplicar esse padrÃ£o a recursos persistentes ou crÃ­ticos.

## 7. Adicionar Placement e seleÃ§Ã£o de clusters

O `Placement` seleciona clusters; ele **nÃ£o cria sozinho um ManifestWork**. Para este laboratÃ³rio, primeiro use o Placement para validar a seleÃ§Ã£o. A criaÃ§Ã£o dos ManifestWork continua declarativa atÃ© que vocÃª implemente uma integraÃ§Ã£o explÃ­cita.

Crie `manifests/02-placement-dev.yaml`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-dev
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            environment: dev
```

Aplique labels nos clusters gerenciados, no Hub:

```bash
kubectl label managedcluster spoke-dev environment=dev --overwrite --context "$CTX_HUB_CLUSTER"
kubectl label managedcluster spoke-prod environment=prod --overwrite --context "$CTX_HUB_CLUSTER"
```

Aplique e confira o Placement:

```bash
kubectl apply -f manifests/02-placement-dev.yaml --context "$CTX_HUB_CLUSTER"
kubectl get placement -n default --context "$CTX_HUB_CLUSTER"
kubectl get placementdecision -n default --context "$CTX_HUB_CLUSTER" -o yaml
```

**CritÃ©rio de aceite:** a decisÃ£o de Placement seleciona `spoke-dev`, mas nÃ£o `spoke-prod`.

> A API e os detalhes de seleÃ§Ã£o podem variar por versÃ£o do OCM. Se o CRD instalado nÃ£o aceitar `cluster.open-cluster-management.io/v1beta1`, consulte a API correspondente Ã  versÃ£o instalada e ajuste o `apiVersion`/schema.

## 8. Instalar Argo CD no Hub

O Argo CD do Hub Ã© responsÃ¡vel por recursos do prÃ³prio Hub e, posteriormente, por manifests declarativos de governanÃ§a e ManifestWork armazenados em Git.

Para o primeiro laboratÃ³rio, instale usando os manifests oficiais. Para uso contÃ­nuo, fixe uma versÃ£o especÃ­fica em vez de usar `stable`.

```bash
kubectl create namespace argocd --context "$CTX_HUB_CLUSTER"
kubectl apply --server-side --context "$CTX_HUB_CLUSTER" \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Aguarde e valide:

```bash
kubectl rollout status deployment/argocd-server -n argocd --context "$CTX_HUB_CLUSTER" --timeout=180s
kubectl get pods -n argocd --context "$CTX_HUB_CLUSTER"
```

A instalaÃ§Ã£o padrÃ£o Ã© suficiente para laboratÃ³rio, mas nÃ£o deve ser considerada uma configuraÃ§Ã£o de produÃ§Ã£o. Em produÃ§Ã£o, avalie versÃ£o fixada, TLS, autenticaÃ§Ã£o/SSO, RBAC, backups, recursos e alta disponibilidade.

## 9. Instalar Argo CD em cada Spoke

### EstratÃ©gia inicial recomendada: instalaÃ§Ã£o explÃ­cita para validar a separaÃ§Ã£o

Instale um Argo CD independente em cada Spoke. Isso valida a premissa de que cada cluster gerencia suas prÃ³prias aplicaÃ§Ãµes antes de automatizar o bootstrap.

```bash
kubectl create namespace argocd --context "$CTX_SPOKE_DEV"
kubectl apply --server-side --context "$CTX_SPOKE_DEV" \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl create namespace argocd --context "$CTX_SPOKE_PROD"
kubectl apply --server-side --context "$CTX_SPOKE_PROD" \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Valide os dois:

```bash
kubectl rollout status deployment/argocd-server -n argocd --context "$CTX_SPOKE_DEV" --timeout=180s
kubectl rollout status deployment/argocd-server -n argocd --context "$CTX_SPOKE_PROD" --timeout=180s
kubectl get pods -n argocd --context "$CTX_SPOKE_DEV"
kubectl get pods -n argocd --context "$CTX_SPOKE_PROD"
```

Neste ponto, a instalaÃ§Ã£o ainda nÃ£o Ã© automatizada pelo Hub. Isso Ã© intencional: primeiro validar o funcionamento de cada camada isoladamente.

## 10. Automatizar o bootstrap do Argo CD nos Spokes via ManifestWork

Depois que o teste de ManifestWork estiver estÃ¡vel, Ã© possÃ­vel usar o Hub para distribuir os manifests do Argo CD. HÃ¡ uma consideraÃ§Ã£o importante: os manifests upstream sÃ£o extensos e podem mudar; nÃ£o Ã© ideal colocar uma URL remota genÃ©rica diretamente como se fosse um Ãºnico recurso dentro do ManifestWork.

### PadrÃ£o recomendado

1. Escolha e fixe uma versÃ£o do Argo CD.
2. Baixe os manifests oficiais dessa versÃ£o para o repositÃ³rio Git.
3. Revise os manifests e ajuste namespace, RBAC, recursos e seguranÃ§a.
4. Gere um conjunto de recursos Kubernetes que o ManifestWork possa transportar.
5. Crie um ManifestWork por Spoke, no namespace que corresponde ao nome do ManagedCluster.
6. Valide o estado do ManifestWork e os pods do Argo CD no destino.

Exemplo de esqueleto para `manifests/03-argocd-bootstrap-spoke-dev.yaml` (o conteÃºdo de `manifests:` deve conter os objetos Kubernetes efetivamente renderizados e revisados; este esqueleto nÃ£o instala Argo CD sozinho):

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: argocd-bootstrap
  namespace: spoke-dev
spec:
  workload:
    manifests:
      # Insira aqui os objetos renderizados e revisados da versÃ£o
      # fixada do Argo CD. Garanta que namespaces, RBAC, CRDs,
      # Deployments, Services e ConfigMaps estejam incluÃ­dos.
      - apiVersion: v1
        kind: Namespace
        metadata:
          name: argocd-bootstrap-placeholder
```

**NÃ£o aplique esse esqueleto como se fosse a instalaÃ§Ã£o real do Argo CD.** Ele serve para mostrar o envelope do ManifestWork. A instalaÃ§Ã£o completa precisa de todos os objetos renderizados, e CRDs/RBAC exigem validaÃ§Ã£o cuidadosa. Para reduzir risco, faÃ§a primeiro o bootstrap manual da seÃ§Ã£o anterior; sÃ³ automatize quando tiver um bundle testado e versionado.

Aplique o bundle validado e acompanhe:

```bash
kubectl apply -f manifests/03-argocd-bootstrap-spoke-dev.yaml --context "$CTX_HUB_CLUSTER"
kubectl describe manifestwork argocd-bootstrap -n spoke-dev --context "$CTX_HUB_CLUSTER"
kubectl get pods -n argocd --context "$CTX_SPOKE_DEV"
```

Repita para `spoke-prod` usando `metadata.namespace: spoke-prod`.

### Alternativa a considerar

Avalie a integraÃ§Ã£o de ciclo de vida de aplicaÃ§Ãµes do OCM com Argo CD, incluindo o add-on/integration de Argo CD disponÃ­vel para a versÃ£o escolhida. Essa integraÃ§Ã£o pode evitar que vocÃª tenha de criar manualmente um mecanismo para gerar ManifestWork a partir de decisÃµes de Placement.

## 11. Instalar OLM: somente se houver Operators necessÃ¡rios

O OLM nÃ£o Ã© obrigatÃ³rio para OCM, ManifestWork ou Argo CD. Instale-o somente se o laboratÃ³rio precisar administrar Operators via catÃ¡logo.

Antes de instalar, decida:
- qual distribuiÃ§Ã£o/versÃ£o do OLM serÃ¡ usada;
- qual catÃ¡logo fornecerÃ¡ o Operator desejado;
- se existe um Operator Argo CD compatÃ­vel e mantido nesse catÃ¡logo;
- como serÃ£o fixadas as versÃµes e auditadas as permissÃµes.

**NÃ£o presuma que OLM fornece automaticamente um Operator oficial do Argo CD.** O mÃ©todo mais direto e previsÃ­vel para este laboratÃ³rio Ã© instalar Argo CD pelos manifests oficiais. Se o objetivo do exercÃ­cio for aprender OLM, faÃ§a isso como uma etapa separada com um Operator conhecido e documentado.

Se optar por instalar OLM via ManifestWork, aplique os manifests oficiais e revisados do release escolhido como um bundle completo. NÃ£o use um URL de `latest` sem revisÃ£o em um ambiente que precise ser reproduzÃ­vel.

## 12. Testar GitOps local em cada Spoke

Crie um repositÃ³rio Git com uma aplicaÃ§Ã£o simples. Exemplo de estrutura:

```text
platform-lab/
â”œâ”€â”€ hub/
â”‚   â”œâ”€â”€ argocd/
â”‚   â””â”€â”€ placements/
â”œâ”€â”€ spokes/
â”‚   â”œâ”€â”€ dev/
â”‚   â”‚   â””â”€â”€ demo-app/
â”‚   â””â”€â”€ prod/
â”‚       â””â”€â”€ demo-app/
â””â”€â”€ governance/
```

Crie `spokes/dev/demo-app/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
```

Crie `spokes/dev/demo-app/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.10.3
          ports:
            - name: http
              containerPort: 80
```

Crie `spokes/dev/demo-app/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: demo-app
spec:
  selector:
    app: whoami
  ports:
    - name: http
      port: 80
      targetPort: http
```

Crie `spokes/dev/demo-app/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
```

No Argo CD local do Spoke, crie uma Application que aponte para o caminho `spokes/dev/demo-app` do repositÃ³rio. Exemplo de `spokes/dev/argocd-application.yaml` (substitua a URL do repositÃ³rio):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/SEU-USUARIO/platform-lab.git
    targetRevision: main
    path: spokes/dev/demo-app
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Aplique a Application no Spoke de desenvolvimento:

```bash
kubectl apply -f spokes/dev/argocd-application.yaml --context "$CTX_SPOKE_DEV"
kubectl get applications -n argocd --context "$CTX_SPOKE_DEV"
kubectl get deployment,service -n demo-app --context "$CTX_SPOKE_DEV"
```

Para produÃ§Ã£o, crie um caminho separado e aplique uma Application equivalente no contexto `kind-spoke-prod`. NÃ£o use o mesmo caminho de desenvolvimento se os parÃ¢metros de produÃ§Ã£o forem diferentes.

**CritÃ©rio de aceite:** o Argo CD local do Spoke sincroniza a aplicaÃ§Ã£o a partir do Git e a aplicaÃ§Ã£o fica saudÃ¡vel.

## 13. Adicionar governanÃ§a

Depois que o registro, a distribuiÃ§Ã£o e o GitOps local estiverem comprovados, adicione governanÃ§a.

OpÃ§Ãµes:
1. **OCM Governance Policy Framework/add-on:** para polÃ­ticas centralizadas e seleÃ§Ã£o por cluster.
2. **ManifestWork:** para distribuir recursos de configuraÃ§Ã£o que tenham um escopo claro.
3. **Argo CD do Hub:** para versionar e reconciliar os manifests centrais de governanÃ§a.

Evite usar `ManifestWork` para administrar um Deployment que jÃ¡ pertence ao Argo CD local. Para governanÃ§a, prefira polÃ­ticas, RBAC, namespaces, quotas e configuraÃ§Ãµes que nÃ£o concorram com os recursos de aplicaÃ§Ã£o.

Antes de aplicar polÃ­ticas de enforcement, teste primeiro em modo de auditoria ou em um cluster descartÃ¡vel. Documente o que ocorre quando um cluster deixa de corresponder ao Placement e quando uma polÃ­tica Ã© removida.

## 14. Checklist de validaÃ§Ã£o por etapa

- [ ] Os trÃªs clusters Kind estÃ£o acessÃ­veis por seus contextos.
- [ ] O OCM Hub estÃ¡ saudÃ¡vel.
- [ ] `spoke-dev` e `spoke-prod` estÃ£o registrados e `AVAILABLE=True`.
- [ ] O ManifestWork cria o ConfigMap no Spoke.
- [ ] AtualizaÃ§Ã£o do ManifestWork Ã© refletida no destino.
- [ ] A remoÃ§Ã£o do ManifestWork tem o comportamento esperado.
- [ ] Placement seleciona somente o cluster com o label correto.
- [ ] Argo CD do Hub estÃ¡ saudÃ¡vel e gerencia recursos do Hub.
- [ ] Cada Spoke tem um Argo CD independente.
- [ ] O Argo CD local sincroniza uma aplicaÃ§Ã£o a partir do Git.
- [ ] A instalaÃ§Ã£o remota do Argo CD foi testada com um bundle fixado, se essa automaÃ§Ã£o for necessÃ¡ria.
- [ ] GovernanÃ§a foi testada sem interferir nas aplicaÃ§Ãµes gerenciadas localmente.
- [ ] HÃ¡ um responsÃ¡vel Ãºnico pela reconciliaÃ§Ã£o de cada objeto.

## 15. Troubleshooting rÃ¡pido

### ManagedCluster nÃ£o fica disponÃ­vel

```bash
kubectl describe managedcluster spoke-dev --context "$CTX_HUB_CLUSTER"
kubectl get pods -A --context "$CTX_SPOKE_DEV" | grep open-cluster-management
kubectl get csr --context "$CTX_HUB_CLUSTER"
```

Confira endpoint, conectividade, token, aprovaÃ§Ã£o do registro e logs dos agentes.

### ManifestWork nÃ£o aplica os recursos

```bash
kubectl describe manifestwork bootstrap-test-configmap -n spoke-dev --context "$CTX_HUB_CLUSTER"
kubectl get events -A --context "$CTX_SPOKE_DEV" --sort-by=.lastTimestamp
```

Confira namespace do ManifestWork (deve corresponder ao nome do ManagedCluster), `apiVersion`, permissÃµes do agente e erros de schema.

### Argo CD nÃ£o sincroniza

```bash
kubectl get applications -n argocd --context "$CTX_SPOKE_DEV"
kubectl get pods -n argocd --context "$CTX_SPOKE_DEV"
kubectl logs -n argocd deployment/argocd-application-controller --context "$CTX_SPOKE_DEV"
kubectl logs -n argocd deployment/argocd-repo-server --context "$CTX_SPOKE_DEV"
```

Confira URL/credenciais do Git, caminho, branch, renderizaÃ§Ã£o Kustomize/Helm e permissÃµes de destino.

## 16. Limpeza do laboratÃ³rio

Remova primeiro as aplicaÃ§Ãµes e ManifestWork de teste. Depois, se quiser apagar tudo, remova os clusters Kind:

```bash
kind delete cluster --name spoke-dev
kind delete cluster --name spoke-prod
kind delete cluster --name hub
```

Isso apaga os clusters locais e seus dados. Confirme que nÃ£o hÃ¡ dados necessÃ¡rios antes de executar.

## 17. ReferÃªncias oficiais

- OCM Quick Start com Kind: https://open-cluster-management.io/docs/getting-started/quick-start/
- InicializaÃ§Ã£o do OCM Hub: https://open-cluster-management.io/docs/getting-started/installation/start-the-control-plane/
- Registro de cluster gerenciado: https://open-cluster-management.io/docs/getting-started/installation/register-a-cluster/
- ManifestWork: https://open-cluster-management.io/concepts/manifestwork/
- InstalaÃ§Ã£o do Argo CD: https://argo-cd.readthedocs.io/en/latest/operator-manual/installation/
- Bootstrap de clusters com Argo CD/ApplicationSet: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
- IntegraÃ§Ã£o OCM e Argo CD: https://open-cluster-management.io/docs/getting-started/integration/app-lifecycle/

---

## SequÃªncia recomendada resumida

1. Criar Hub e Spokes no Kind.
2. Inicializar OCM no Hub.
3. Registrar e aprovar os Spokes.
4. Provar ManifestWork com ConfigMap.
5. Validar Placement e seleÃ§Ã£o.
6. Instalar Argo CD no Hub.
7. Instalar Argo CD independente nos Spokes.
8. Provar GitOps local em cada Spoke.
9. Automatizar o bootstrap do Argo CD via ManifestWork, se necessÃ¡rio.
10. Adicionar governanÃ§a OCM e validar polÃ­ticas.
11. SÃ³ incluir OLM se houver Operators concretos para administrar.

**RecomendaÃ§Ã£o central:** nÃ£o comece automatizando OLM e Argo CD de uma sÃ³ vez. Primeiro prove registro OCM â†’ ManifestWork â†’ recurso no Spoke. Depois automatize cada camada de bootstrap separadamente. Isso torna falhas rastreÃ¡veis e mantÃ©m clara a propriedade dos recursos.
# Microsserviços no Azure com AKS — Guia completo

Nov 21, 2026 · @Oi

## 1. Introdução

Este guia ensina a projetar, construir e operar uma aplicação de **microsserviços** no **Azure Kubernetes Service (AKS)**, do desenho dos serviços até a operação em produção com observabilidade, segurança e GitOps. Ele assume que você já conhece os fundamentos de Kubernetes (Pods, Deployments, Services, Ingress) e de contêineres.

### 1.1 Microsserviços: o que são e quando valem a pena

Uma arquitetura de **microsserviços** divide uma aplicação em serviços pequenos e independentes, cada um responsável por uma capacidade de negócio, com seus próprios dados, implantado e escalado separadamente, e mantido por um time que é dono dele de ponta a ponta.

| Aspecto | Monólito | Microsserviços |
| --- | --- | --- |
| Implantação | Uma unidade; qualquer mudança reimplanta tudo | Cada serviço é implantado de forma independente |
| Escala | A aplicação inteira escala junta | Cada serviço escala conforme a própria demanda |
| Dados | Um banco compartilhado | Cada serviço é dono dos seus dados |
| Tecnologia | Uma stack | Cada serviço pode usar a stack mais adequada |
| Falhas | Uma falha pode derrubar tudo | Falhas podem ser isoladas, se o design ajudar |
| Complexidade | No código | Na rede, na operação e na consistência de dados |
| Times | Coordenação constante | Times autônomos com fronteiras claras |

> **Atenção:** microsserviços trocam complexidade de código por complexidade distribuída. Para um time pequeno, um produto novo ou um domínio ainda pouco entendido, um **monólito modular** bem estruturado costuma ser a melhor escolha. Migre para microsserviços quando houver necessidades concretas: times que se bloqueiam, partes com perfis de escala muito diferentes ou ciclos de entrega que não podem esperar uns pelos outros.

### 1.2 Por que AKS

O **AKS** é o Kubernetes gerenciado do Azure. A Microsoft opera o control plane (API server, etcd, scheduler), aplica atualizações de segurança e integra o cluster aos serviços do Azure: identidade (Microsoft Entra ID), rede (VNet, Application Gateway), registro (Azure Container Registry), monitoramento (Azure Monitor) e políticas (Azure Policy). Você se concentra nos *node pools* e nas aplicações.

| Opção no Azure | Quando escolher |
| --- | --- |
| **AKS** | Muitos serviços, controle total do Kubernetes, necessidades de rede, segurança ou workloads específicos (GPUs, stateful), time com experiência em Kubernetes |
| **Azure Container Apps** | Microsserviços e jobs sem gerenciar Kubernetes: escala a zero, Dapr e KEDA embutidos; menos controle |
| **Azure App Service** | Aplicações web e APIs tradicionais, poucos serviços |
| **Azure Functions** | Lógica orientada a eventos, de curta duração |

Muitas empresas combinam: AKS para o núcleo da plataforma, Container Apps ou Functions para serviços periféricos e orientados a eventos.

### 1.3 O caso prático: Loja Online

Todos os capítulos constroem a mesma aplicação, uma **loja online** com seis microsserviços:

| Serviço | Responsabilidade | Dados | Expõe |
| --- | --- | --- | --- |
| `catalogo` | Produtos, preços e busca | Cosmos DB | API REST |
| `carrinho` | Carrinho de compras do cliente | Azure Cache for Redis | API REST |
| `pedidos` | Criação e ciclo de vida dos pedidos | Azure SQL Database | API REST e eventos |
| `pagamentos` | Integração com o provedor de pagamento | Azure Database for PostgreSQL | Eventos |
| `estoque` | Reserva e baixa de estoque | Azure Database for PostgreSQL | Eventos |
| `notificacoes` | E-mails e mensagens ao cliente | — | Consome eventos |

O fluxo de compra cruza vários serviços e por isso exercita os problemas reais de microsserviços: consistência entre bancos diferentes, falhas parciais, mensageria, rastreamento distribuído e implantação independente.

### 1.4 Pré-requisitos e como usar o guia

- Uma assinatura do Azure com permissão para criar recursos (as contas gratuitas servem para boa parte dos laboratórios).
- **Azure CLI** (`az`), **kubectl**, **Helm**, **Docker** e **Git** instalados.
- Noções de Kubernetes e de uma linguagem de back-end; os exemplos usam .NET, Java e Node.js.

> **Custos:** clusters AKS, Application Gateway, API Management e bancos gerenciados cobram enquanto existirem. Crie todos os recursos de um laboratório em um mesmo resource group e apague-o ao final com `az group delete --name <rg> --yes --no-wait`. Configure um alerta de orçamento antes de começar.

Este documento pode ser **exportado para PDF e Markdown** (além de Word e Google Docs) pela opção de exportação do próprio documento.

## 2. Arquitetura de referência

A requisição do cliente passa por três camadas públicas no máximo (borda, API Management e o gateway de entrada); todo o resto — cluster, bancos e mensageria — fica em uma VNet privada, acessado por Private Endpoints.

&#91;embedded content: Arquitetura de referência da Loja Online no Azure\]

### 2.1 Os componentes e o papel de cada um

| Camada | Serviço | Papel | Capítulo |
| --- | --- | --- | --- |
| Borda | Azure Front Door + WAF | CDN, TLS, roteamento global, proteção contra ataques web e DDoS | 8 |
| APIs | Azure API Management | Contrato público das APIs, autenticação, limites de uso, versões, portal do desenvolvedor | 8 |
| Entrada no cluster | App routing com Gateway API ou Application Gateway for Containers | Roteamento HTTP para os serviços dentro do cluster | 8 |
| Computação | AKS | Executa os microsserviços | 5 a 7 |
| Dados | Cosmos DB, Azure Cache for Redis, Azure SQL, PostgreSQL Flexible Server | Um banco por serviço, cada um com o modelo de dados adequado | 10 |
| Mensageria | Azure Service Bus (e Event Hubs para streaming) | Comunicação assíncrona e eventos entre serviços | 9 |
| Imagens | Azure Container Registry | Registro privado das imagens de contêiner | 6 |
| Segredos | Azure Key Vault | Segredos, certificados e chaves | 11 |
| Identidade | Microsoft Entra ID + Workload Identity | Acesso de pessoas ao cluster e dos Pods aos serviços do Azure, sem senhas | 11 |
| Observabilidade | Azure Monitor (Managed Prometheus, Managed Grafana, Container Insights, Application Insights) | Métricas, logs e rastreamento distribuído | 13 |
| Governança | Azure Policy, Defender for Containers | Regras obrigatórias e proteção de ameaças | 15 |

### 2.2 Decisões de arquitetura iniciais

| Decisão | Recomendação padrão | Quando mudar |
| --- | --- | --- |
| Modo do AKS | **AKS Automatic** para times que querem padrões seguros prontos; **AKS Standard** para controle fino | Requisitos específicos de rede, de node pools ou de add-ons |
| Rede | **Azure CNI Overlay** com dataplane **Cilium** | Pods que precisam de IPs roteados na VNet |
| Entrada | **Gateway API**, pelo app routing add-on ou pelo Application Gateway for Containers | Necessidades que o Gateway API ainda não cobre |
| Comunicação | REST/gRPC para consultas; **eventos no Service Bus** para tudo que pode ser assíncrono | — |
| Dados | Um banco **gerenciado** por serviço, fora do cluster | Caches efêmeros ou workloads stateful específicas |
| Implantação | **GitOps** com Flux (extensão do AKS) ou Argo CD | — |
| Infraestrutura | **Terraform** ou **Bicep**, nunca pelo portal | — |

Registre cada decisão importante em um **ADR** (*Architecture Decision Record*): um arquivo curto no repositório com contexto, decisão, alternativas e consequências.

### 2.3 Região

Para usuários no Brasil, a região natural é **Brazil South** (`brazilsouth`, São Paulo), que tem zonas de disponibilidade. Confira se todos os serviços e tamanhos de VM que você pretende usar estão disponíveis nela; para recuperação de desastres, defina também uma região secundária (capítulo 16).

## 3. Design de microsserviços

A parte mais difícil de microsserviços não é o Kubernetes: é decidir **onde ficam as fronteiras**. Fronteiras erradas geram um “monólito distribuído”, em que serviços precisam ser alterados e implantados juntos e falham juntos.

### 3.1 Encontrando as fronteiras com DDD

O **Domain-Driven Design (DDD)** ajuda a dividir o sistema por capacidades de negócio:

- **Bounded context:** uma área do negócio com linguagem e regras próprias. “Produto” no catálogo (descrição, fotos, preço) é diferente de “produto” no estoque (quantidade, depósito). Cada bounded context é um bom candidato a serviço.
- **Agregado:** um conjunto de objetos que muda junto e mantém suas regras consistentes, como um `Pedido` e seus itens. Uma transação nunca deve atravessar dois agregados de serviços diferentes.
- **Event storming:** oficina em que negócio e tecnologia listam os eventos do domínio (`PedidoCriado`, `PagamentoAprovado`, `EstoqueReservado`) e os agrupam; os grupos revelam os contextos.

Sinais de uma boa fronteira: o serviço tem um dono claro, pode ser implantado sozinho, conversa pouco com os outros e não compartilha tabelas.

### 3.2 Banco de dados por serviço

Cada serviço é o **único dono** dos seus dados. Nenhum outro serviço lê ou escreve diretamente no banco dele; o acesso é sempre pela API ou por eventos. Isso permite mudar o esquema, a tecnologia e a escala de cada banco sem coordenar com outros times.

O preço é que não existem mais *joins* nem transações entre serviços. Os padrões a seguir resolvem isso.

### 3.3 Comunicação síncrona e assíncrona

|  | Síncrona (REST, gRPC) | Assíncrona (mensagens e eventos) |
| --- | --- | --- |
| Como funciona | O chamador espera a resposta | O emissor publica e segue; o consumidor processa quando puder |
| Acoplamento | Temporal: os dois precisam estar no ar | Baixo: o consumidor pode estar fora do ar por um tempo |
| Bom para | Consultas que precisam de resposta imediata | Comandos e notificações de mudança de estado |
| Risco | Falhas em cascata e latência somada | Consistência eventual, mensagens duplicadas, depuração mais difícil |
| No Azure | Services do Kubernetes, Gateway API, API Management | Service Bus (filas e tópicos), Event Hubs, Event Grid |

Regra prática: **leia de forma síncrona e mude estado de forma assíncrona.** Se uma requisição de usuário precisa encadear chamadas síncronas por quatro serviços, a disponibilidade dela é o produto das quatro, e as fronteiras provavelmente estão erradas.

### 3.4 Saga: transações entre serviços

Uma **saga** é uma sequência de transações locais, uma por serviço, ligadas por eventos. Se um passo falha, os anteriores são desfeitos com **ações de compensação**. O fluxo de compra da Loja:

1. `pedidos` cria o pedido como `PENDENTE` e publica `PedidoCriado`.
2. `estoque` reserva os itens e publica `EstoqueReservado` (ou `EstoqueIndisponivel`).
3. `pagamentos` cobra e publica `PagamentoAprovado` (ou `PagamentoRecusado`).
4. `pedidos` marca o pedido como `CONFIRMADO`; `notificacoes` avisa o cliente.
5. Se o pagamento for recusado, `estoque` libera a reserva (compensação) e `pedidos` marca o pedido como `CANCELADO`.

| Estilo | Como funciona | Quando usar |
| --- | --- | --- |
| Coreografia | Cada serviço reage a eventos dos outros; não há coordenador | Fluxos curtos, com poucos passos |
| Orquestração | Um orquestrador (por exemplo, no serviço `pedidos` ou com Dapr Workflow / Durable Functions) envia comandos e controla o estado | Fluxos longos, com muitas regras e compensações |

### 3.5 Outbox transacional

O serviço precisa gravar no banco **e** publicar o evento. Se gravar e cair antes de publicar, o evento se perde; se publicar e a gravação falhar, o evento é falso. O padrão **outbox** resolve: o evento é gravado em uma tabela `outbox` na **mesma transação** do dado, e um processo separado lê a tabela e publica no Service Bus.

```sql
BEGIN TRANSACTION;
  INSERT INTO pedidos (id, cliente_id, status, total) VALUES (@id, @cliente, 'PENDENTE', @total);
  INSERT INTO outbox (id, tipo, payload, criado_em)
    VALUES (NEWID(), 'PedidoCriado', @json, SYSUTCDATETIME());
COMMIT;
```

Como o publicador pode enviar o mesmo evento mais de uma vez, os consumidores devem ser **idempotentes**: guardam o id das mensagens já processadas e ignoram repetições.

### 3.6 Outros padrões essenciais

| Padrão | Problema que resolve | Implementação no Azure |
| --- | --- | --- |
| API Gateway | Clientes não devem conhecer cada serviço | API Management, Gateway API |
| Backend for Frontend (BFF) | Web e app móvel precisam de respostas diferentes | Um serviço de agregação por tipo de cliente |
| CQRS | Consultas complexas que atravessam serviços | Um modelo de leitura alimentado por eventos (por exemplo, no Cosmos DB ou no Azure AI Search) |
| Circuit breaker, retry, timeout | Falhas em cascata | Polly (.NET), Resilience4j (Java), políticas do Dapr ou do Istio |
| Strangler fig | Migrar um monólito aos poucos | Roteamento no API Management ou no gateway, serviço a serviço |
| Health check | Detectar instâncias com problema | Probes do Kubernetes (capítulo 7) |

### 3.7 Contratos e versionamento de APIs

- Descreva APIs REST com **OpenAPI** e eventos com **AsyncAPI** ou CloudEvents, versionados no repositório do serviço.
- Mudanças **compatíveis** (campos novos e opcionais) não exigem nova versão; mudanças incompatíveis exigem `v2`, com as duas versões convivendo por um período.
- Teste contratos automaticamente entre produtor e consumidor (por exemplo, com Pact) para que um serviço não quebre outro ao ser implantado.

### Laboratório

- [ ] Faça um event storming da Loja Online e confira se os bounded contexts batem com os seis serviços do capítulo 1.
- [ ] Escreva a especificação OpenAPI do serviço `pedidos` e os eventos `PedidoCriado`, `PagamentoAprovado` e `PagamentoRecusado` no formato CloudEvents.
- [ ] Desenhe a saga de compra com as compensações e decida entre coreografia e orquestração, registrando a escolha em um ADR.

## 4. Fundação no Azure

Antes de criar o cluster, organize as assinaturas, a identidade e a rede. Essas decisões são difíceis de mudar depois: trocar o plano de endereçamento de uma VNet com um cluster em produção, por exemplo, exige recriar o cluster.

### 4.1 Hierarquia de recursos

| Nível | Uso recomendado |
| --- | --- |
| Management groups | Agrupam assinaturas e herdam políticas (por exemplo, `Plataforma`, `Workloads/Produção`, `Workloads/Não produção`) |
| Subscriptions | Fronteira de faturamento, cotas e acesso. Use assinaturas separadas para produção e não produção |
| Resource groups | Agrupam recursos com o mesmo ciclo de vida: `rg-loja-rede-prd`, `rg-loja-aks-prd`, `rg-loja-dados-prd` |
| Tags | `ambiente`, `aplicacao`, `time`, `centro-de-custo` em todos os recursos |

Em empresas, essa estrutura vem de uma **Azure Landing Zone**, a arquitetura de referência da Microsoft (Cloud Adoption Framework) com management groups, rede hub-and-spoke, políticas e identidade já definidos. O cluster vive em uma **landing zone de aplicação** (uma assinatura *spoke*).

### 4.2 Identidade com Microsoft Entra ID

- **Pessoas:** acessam o Azure e o cluster com contas do Entra ID, por **grupos** (`grp-loja-aks-admins`, `grp-loja-devs`), com MFA e acesso condicional.
- **Cluster:** usa **identidades gerenciadas** para criar recursos do Azure (discos, load balancers), nunca *service principals* com segredo.
- **Pods:** usam **Workload Identity** para acessar Key Vault, Service Bus e bancos sem senhas (capítulo 11).
- **CI/CD:** usa **federação OIDC** com o GitHub ou o Azure DevOps (capítulo 14).
- **Acesso privilegiado:** use o **Privileged Identity Management (PIM)** para elevar permissões em produção só quando necessário e com prazo.

### 4.3 Rede

A topologia mais comum é **hub-and-spoke**: uma VNet *hub* com firewall, VPN ou ExpressRoute e DNS, e uma VNet *spoke* por aplicação, conectadas por peering.

| Sub-rede no spoke | Tamanho sugerido | Uso |
| --- | --- | --- |
| `snet-aks-nodes` | /22 | Nós do AKS (com CNI Overlay, só os nós consomem IPs da VNet) |
| `snet-agc` | /24 | Application Gateway for Containers (sub-rede delegada) |
| `snet-private-endpoints` | /24 | Private Endpoints dos bancos, Key Vault, ACR e Service Bus |
| `snet-apim` | /27 | API Management injetado na VNet, se usado |

Com **Azure CNI Overlay**, os Pods recebem IPs de um espaço privado separado (por exemplo, `10.244.0.0/16`), que não consome endereços da VNet; isso evita o problema clássico de esgotar a sub-rede quando o cluster cresce. Escolha faixas de Pods e de Services que **não se sobreponham** a nenhuma rede on-premises ou de outras VNets conectadas.

| Modelo de rede | IP dos Pods | Quando usar |
| --- | --- | --- |
| **Azure CNI Overlay** | Espaço privado do cluster | Padrão recomendado para a maioria dos clusters |
| Azure CNI com sub-rede para Pods | IPs da VNet | Quando outros sistemas precisam alcançar os Pods diretamente |
| Kubenet | — | Legado; não use em clusters novos |

Combine com o dataplane **Azure CNI powered by Cilium**, que traz desempenho melhor, network policies com eBPF e é pré-requisito de recursos como o node auto-provisioning.

Para o tráfego de saída, use um **NAT Gateway** (IP de saída fixo e escalável) ou roteie pelo **Azure Firewall** do hub (`outboundType: userDefinedRouting`), quando a política de segurança exigir inspeção.

### 4.4 Infraestrutura como código

Crie tudo com **Terraform** ou **Bicep**. Um trecho em Terraform para a rede e o cluster:

```hcl
resource "azurerm_resource_group" "aks" {
  name     = "rg-loja-aks-dev"
  location = "brazilsouth"
  tags     = { ambiente = "dev", aplicacao = "loja" }
}

resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-loja-dev"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  address_space       = ["10.20.0.0/16"]
}

resource "azurerm_subnet" "nodes" {
  name                 = "snet-aks-nodes"
  resource_group_name  = azurerm_resource_group.aks.name
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = ["10.20.0.0/22"]
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                      = "aks-loja-dev"
  location                  = azurerm_resource_group.aks.location
  resource_group_name       = azurerm_resource_group.aks.name
  dns_prefix                = "aks-loja-dev"
  sku_tier                  = "Standard"          # SLA do control plane
  oidc_issuer_enabled       = true                # necessário para Workload Identity
  workload_identity_enabled = true
  local_account_disabled    = true                # só acesso via Entra ID
  automatic_upgrade_channel = "patch"
  node_os_upgrade_channel   = "NodeImage"

  default_node_pool {
    name                         = "system"
    vm_size                      = "Standard_D4ds_v5"
    vnet_subnet_id               = azurerm_subnet.nodes.id
    zones                        = ["1", "2", "3"]
    auto_scaling_enabled         = true
    min_count                    = 2
    max_count                    = 4
    only_critical_addons_enabled = true           # reserva o pool para componentes do sistema
  }

  identity { type = "SystemAssigned" }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled     = true
    admin_group_object_ids = [var.grupo_admins_aks]
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_data_plane  = "cilium"
    network_policy      = "cilium"
    pod_cidr            = "10.244.0.0/16"
    service_cidr        = "10.0.0.0/16"
    dns_service_ip      = "10.0.0.10"
  }
}
```

Os nomes de argumentos do provider `azurerm` mudam entre versões principais: fixe a versão do provider e confira a documentação correspondente. Guarde o estado do Terraform em uma conta de armazenamento com versionamento e acesso restrito.

### 4.5 Controle de custos desde o início

- Crie **orçamentos com alertas** por assinatura e por resource group no Cost Management.
- Ative a **análise de custos do AKS**, que distribui o custo do cluster por namespace.
- Desligue clusters de desenvolvimento fora do horário com `az aks stop` e `az aks start`.
- Use **VMs spot** em node pools de workloads toleráveis a interrupção e **reservas** ou **Savings Plans** para a capacidade estável de produção.

### Laboratório

- [ ] Crie os grupos `grp-loja-aks-admins` e `grp-loja-devs` no Entra ID.
- [ ] Escreva em Terraform (ou Bicep) o resource group, a VNet e as sub-redes da tabela 4.3.
- [ ] Planeje as faixas de IP de VNet, Pods e Services e confirme que não se sobrepõem.
- [ ] Crie um orçamento de baixo valor com alerta por e-mail para os laboratórios.

## 5. Criando o cluster AKS

O AKS tem dois modos de operação. Escolha antes de criar, porque eles mudam quanta configuração você assume.

|  | AKS Automatic | AKS Standard |
| --- | --- | --- |
| Proposta | Cluster pronto para produção com padrões opinativos | Controle total sobre a configuração |
| Nós | Provisionados automaticamente (node auto-provisioning, baseado em Karpenter) | Node pools que você define e escala |
| Já vem habilitado | Rede Overlay com Cilium, Workload Identity, Managed Prometheus, KEDA, upgrades automáticos, *deployment safeguards*, entrada por Gateway API (em clusters 1.36+) | Você habilita o que precisar |
| Para quem | Times que querem focar nas aplicações | Times de plataforma com requisitos específicos |

O **AKS Automatic** está disponível de forma geral desde 2025 ([anúncio](https://azure.microsoft.com/blog/azure-kubernetes-service-automatic-fast-and-frictionless-kubernetes-for-all/)), e o node auto-provisioning também pode ser usado em clusters Standard. Este guia usa comandos do modo Standard para mostrar cada peça; no Automatic, muitas delas já vêm prontas.

### 5.1 Versões do Kubernetes

O AKS suporta três versões menores GA do Kubernetes ao mesmo tempo, cada uma por cerca de 12 meses, além de versões **LTS** com suporte estendido para quem habilita esse plano. Em novembro de 2026, a 1.37 é a versão GA mais recente (desde outubro de 2026) e a 1.38 está em preview ([calendário de versões](https://learn.microsoft.com/azure/aks/supported-kubernetes-versions)). Liste as versões da sua região antes de criar:

```bash
az aks get-versions --location brazilsouth --output table
```

### 5.2 Criando um cluster Standard pela CLI

```bash
export RG=rg-loja-aks-dev
export CLUSTER=aks-loja-dev
export LOCATION=brazilsouth
export SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name vnet-loja-dev \
  -n snet-aks-nodes --query id -o tsv)

az aks create \
  --resource-group $RG --name $CLUSTER --location $LOCATION \
  --tier standard \
  --kubernetes-version 1.37 \
  --network-plugin azure --network-plugin-mode overlay \
  --network-dataplane cilium --network-policy cilium \
  --pod-cidr 10.244.0.0/16 \
  --vnet-subnet-id $SUBNET_ID \
  --nodepool-name system --node-vm-size Standard_D4ds_v5 \
  --node-count 2 --zones 1 2 3 \
  --enable-cluster-autoscaler --min-count 2 --max-count 4 \
  --os-sku AzureLinux \
  --enable-managed-identity \
  --enable-aad --enable-azure-rbac --aad-admin-group-object-ids $GRUPO_ADMINS \
  --disable-local-accounts \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-azure-monitor-metrics \
  --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage \
  --generate-ssh-keys

az aks get-credentials --resource-group $RG --name $CLUSTER
kubectl get nodes -o wide
```

Para um cluster Automatic, o comando é bem mais curto: `az aks create --resource-group $RG --name $CLUSTER --sku automatic`.

| Opção | Por quê |
| --- | --- |
| `--tier standard` | SLA financeiro para o control plane; use em produção. O tier `free` serve para testes |
| `--zones 1 2 3` | Distribui os nós entre zonas de disponibilidade |
| `--os-sku AzureLinux` | Sistema dos nós mantido pela Microsoft e otimizado para contêineres (alternativa: Ubuntu) |
| `--disable-local-accounts` | Obriga o login pelo Entra ID, com auditoria |
| `--enable-workload-identity` | Pods acessam o Azure sem segredos (capítulo 11) |
| `--enable-azure-monitor-metrics` | Managed Prometheus (capítulo 13) |

### 5.3 Node pools

Separe o **pool de sistema** (CoreDNS, metrics-server, agentes) dos **pools de usuário** (suas aplicações):

```bash
# pool geral para os microsserviços
az aks nodepool add -g $RG --cluster-name $CLUSTER --name apps \
  --mode User --node-vm-size Standard_D8ds_v5 --zones 1 2 3 \
  --enable-cluster-autoscaler --min-count 2 --max-count 10 \
  --os-sku AzureLinux --labels carga=geral

# pool spot para jobs que toleram interrupção
az aks nodepool add -g $RG --cluster-name $CLUSTER --name spot \
  --mode User --priority Spot --eviction-policy Delete --spot-max-price -1 \
  --node-vm-size Standard_D8ds_v5 --enable-cluster-autoscaler --min-count 0 --max-count 10 \
  --labels carga=batch
```

Pods só vão para o pool spot se tolerarem o *taint* `kubernetes.azure.com/scalesetpriority=spot:NoSchedule` que o AKS aplica a esses nós. Use `nodeSelector` ou *affinity* com os labels para direcionar cada workload ao pool certo.

### 5.4 Upgrades

Um cluster precisa de dois tipos de atualização: da **versão do Kubernetes** e da **imagem do sistema dos nós** (patches de segurança).

| Canal | O que faz |
| --- | --- |
| `--auto-upgrade-channel patch` | Aplica automaticamente patches da mesma versão menor (por exemplo, 1.37.x) |
| `--auto-upgrade-channel stable` | Também sobe de versão menor, mantendo-se uma atrás da mais nova |
| `--node-os-upgrade-channel NodeImage` | Troca periodicamente a imagem dos nós pela mais recente |

Defina uma **janela de manutenção** para que essas atualizações ocorram em horário de baixo uso:

```bash
az aks maintenanceconfiguration add -g $RG --cluster-name $CLUSTER \
  --name aksManagedAutoUpgradeSchedule \
  --schedule-type Weekly --day-of-week Sunday --interval-weeks 1 \
  --start-time 03:00 --duration 4 --utc-offset -03:00
```

Para que upgrades não derrubem as aplicações, cada serviço precisa de réplicas em zonas diferentes, probes corretas e um **PodDisruptionBudget** (capítulo 7). Antes de subir de versão menor, leia a seção de *breaking changes* do calendário de versões e procure APIs depreciadas no painel de diagnóstico do cluster.

### 5.5 Acesso ao cluster

Com o Azure RBAC para Kubernetes, as permissões dentro do cluster também são papéis do Azure:

| Papel do Azure | Permite |
| --- | --- |
| Azure Kubernetes Service RBAC Cluster Admin | Tudo no cluster |
| Azure Kubernetes Service RBAC Admin | Administrar um namespace |
| Azure Kubernetes Service RBAC Writer | Criar e alterar recursos de um namespace |
| Azure Kubernetes Service RBAC Reader | Ler recursos de um namespace (sem segredos) |

```bash
az role assignment create --role "Azure Kubernetes Service RBAC Writer" \
  --assignee $GRUPO_DEVS \
  --scope $(az aks show -g $RG -n $CLUSTER --query id -o tsv)/namespaces/pedidos
```

O plugin `kubelogin` faz a autenticação do `kubectl` com o Entra ID. Para restringir ainda mais, crie um **cluster privado** (API server sem IP público) ou limite os IPs autorizados com `--api-server-authorized-ip-ranges`.

### Laboratório

- [ ] Crie o cluster de desenvolvimento com o comando da seção 5.2 (ou com o Terraform do capítulo 4).
- [ ] Adicione os pools `apps` e `spot` e confirme os labels e taints com `kubectl get nodes --show-labels`.
- [ ] Configure a janela de manutenção e o canal de upgrade de patches.
- [ ] Dê ao grupo de desenvolvedores o papel Writer apenas no namespace `pedidos` e teste com uma conta do grupo.

## 6. Contêineres e Azure Container Registry

Cada microsserviço é entregue como uma **imagem de contêiner** imutável, versionada e armazenada em um registro privado. Boas imagens são pequenas, rodam sem root, não têm segredos e são construídas sempre pelo pipeline, nunca na máquina de um desenvolvedor.

### 6.1 Dockerfiles multi-stage

O *build* multi-stage compila em uma imagem com SDK e copia só o resultado para uma imagem de runtime enxuta.

**.NET (serviço `pedidos`):**

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
# APP_UID: usuário não root já definido na imagem oficial
USER $APP_UID
ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "Pedidos.Api.dll"]
```

**Java com Spring Boot (serviço `estoque`):**

```dockerfile
FROM eclipse-temurin:25-jdk AS build
WORKDIR /src
COPY . .
RUN ./mvnw -q package -DskipTests

FROM eclipse-temurin:25-jre
RUN useradd --system --uid 10001 app
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
USER 10001
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Node.js (serviço `notificacoes`):**

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/main.js"]
```

Boas práticas para todas as imagens:

- Fixe versões das imagens base (nunca apenas `latest`) e atualize-as com frequência, por exemplo com o Dependabot ou o Renovate.
- Use um `.dockerignore` para não copiar `.git`, segredos, `node_modules` ou artefatos locais.
- Rode como usuário não root e escute em uma porta acima de 1024 (8080).
- Escreva logs na saída padrão (stdout), em JSON; o cluster coleta de lá.
- Trate o sinal `SIGTERM` para encerrar com graça: pare de aceitar requisições, termine as que estão em andamento e feche conexões.

### 6.2 Azure Container Registry

```bash
export ACR=acrlojadev$RANDOM        # nome globalmente único, só letras e números
az acr create -g $RG -n $ACR --sku Premium --location brazilsouth

# permite que o cluster baixe imagens (concede AcrPull à identidade do kubelet)
az aks update -g $RG -n $CLUSTER --attach-acr $ACR

# build na nuvem, sem Docker local
az acr build -r $ACR -t pedidos:$(git rev-parse --short HEAD) ./src/pedidos
```

| SKU | Quando usar |
| --- | --- |
| Basic | Laboratórios e projetos pequenos |
| Standard | A maioria dos ambientes de desenvolvimento |
| Premium | Produção: Private Endpoint, geo-replicação, mais armazenamento e throughput, regras de rede |

Em produção, desabilite o acesso público do registro e use **Private Endpoint**; desabilite também o usuário administrador (`--admin-enabled false`), porque toda autenticação deve ser por identidade do Entra ID.

### 6.3 Tags e promoção de imagens

- **Tag imutável** por build: o hash do commit (`pedidos:a1b2c3d`) ou a versão semântica (`pedidos:1.4.2`).
- Nos manifestos de produção, prefira referenciar pelo **digest** (`pedidos@sha256:...`), que garante exatamente a imagem testada.
- Habilite o **bloqueio de tags** em produção para impedir que uma tag seja sobrescrita.
- Promova a mesma imagem entre registros de ambientes com `az acr import`, sem reconstruir.
- Limpe imagens antigas com o `acr purge` em uma ACR Task agendada.

### 6.4 Segurança da cadeia de suprimentos

| Prática | Ferramenta |
| --- | --- |
| Varredura de vulnerabilidades no registro e em execução | Microsoft Defender for Containers |
| Varredura no CI, antes do push | Trivy ou Grype, bloqueando falhas críticas |
| SBOM (lista de componentes da imagem) | Syft ou o próprio Docker BuildKit |
| Assinatura de imagens | Notation (Notary Project) com certificados no Key Vault |
| Aceitar só imagens assinadas e de registros confiáveis | Azure Policy com Ratify ou Image Integrity |

### Laboratório

- [ ] Escreva o Dockerfile de dois serviços da Loja em linguagens diferentes e meça o tamanho das imagens com e sem multi-stage.
- [ ] Crie o ACR, vincule-o ao cluster e construa as imagens com `az acr build`.
- [ ] Rode o Trivy sobre uma imagem e corrija ao menos uma vulnerabilidade atualizando a imagem base.
- [ ] Confirme que o contêiner roda como usuário não root com `kubectl exec ... -- id`.

## 7. Implantando os serviços

Cada microsserviço vira um conjunto pequeno e padronizado de objetos do Kubernetes. Padronizar é o que permite que dezenas de serviços sejam operados por poucos times: todos têm probes, recursos, segurança e disponibilidade configurados da mesma forma.

### 7.1 Namespaces e limites

Use **um namespace por serviço** (ou por time), o que separa permissões, cotas e políticas de rede:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pedidos
  labels:
    time: checkout
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cota
  namespace: pedidos
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.memory: 32Gi
```

### 7.2 O Deployment padrão de um serviço

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pedidos-api
  namespace: pedidos
  labels: { app.kubernetes.io/name: pedidos-api, app.kubernetes.io/part-of: loja }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  selector:
    matchLabels: { app.kubernetes.io/name: pedidos-api }
  template:
    metadata:
      labels:
        app.kubernetes.io/name: pedidos-api
        azure.workload.identity/use: "true"          # capítulo 11
    spec:
      serviceAccountName: pedidos-api
      nodeSelector: { carga: geral }
      topologySpreadConstraints:                       # espalha as réplicas entre zonas
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app.kubernetes.io/name: pedidos-api }
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      terminationGracePeriodSeconds: 30
      containers:
        - name: api
          image: acrlojadev.azurecr.io/pedidos@sha256:...   # digest gerado pelo CI
          ports: [{ name: http, containerPort: 8080 }]
          envFrom:
            - configMapRef: { name: pedidos-config }
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits: { memory: 512Mi }
          startupProbe:
            httpGet: { path: /health/startup, port: http }
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet: { path: /health/ready, port: http }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /health/live, port: http }
            periodSeconds: 10
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts: [{ name: tmp, mountPath: /tmp }]
      volumes: [{ name: tmp, emptyDir: {} }]
```

| Configuração | Por quê |
| --- | --- |
| `replicas: 3` + `topologySpreadConstraints` | Uma réplica por zona: a queda de uma zona não derruba o serviço |
| `maxUnavailable: 0` | Atualizações nunca reduzem a capacidade |
| `startupProbe` | Dá tempo para serviços lentos iniciarem (comum em Java) sem reiniciá-los |
| `readinessProbe` | Tira o Pod do Service enquanto ele não consegue atender (por exemplo, sem conexão com o banco) |
| `livenessProbe` | Reinicia o contêiner se o processo travar; nunca deve depender de serviços externos |
| `requests` de CPU e memória | Base para o agendamento e para o autoscaling |
| `limits` só de memória | Limitar CPU costuma causar *throttling* e latência; limitar memória evita que um Pod derrube o nó |
| `readOnlyRootFilesystem` e `drop: [ALL]` | Reduzem o que um invasor consegue fazer dentro do contêiner |

### 7.3 Service, PodDisruptionBudget e configuração

```yaml
apiVersion: v1
kind: Service
metadata: { name: pedidos-api, namespace: pedidos }
spec:
  selector: { app.kubernetes.io/name: pedidos-api }
  ports: [{ name: http, port: 80, targetPort: http }]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: pedidos-api, namespace: pedidos }
spec:
  minAvailable: 2               # upgrades de nós nunca derrubam mais de uma réplica por vez
  selector:
    matchLabels: { app.kubernetes.io/name: pedidos-api }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: pedidos-config, namespace: pedidos }
data:
  ASPNETCORE_ENVIRONMENT: Production
  ServiceBus__Namespace: sb-loja-dev.servicebus.windows.net
  Estoque__BaseUrl: http://estoque-api.estoque.svc.cluster.local
```

Configuração não sensível fica em ConfigMaps; segredos ficam no Key Vault (capítulo 11). Serviços se encontram pelo DNS interno: `<serviço>.<namespace>.svc.cluster.local`.

### 7.4 Empacotando com Helm ou Kustomize

Com seis serviços e três ambientes, copiar YAML não escala. Duas abordagens comuns:

|  | Helm | Kustomize |
| --- | --- | --- |
| Modelo | Templates com valores (`values-dev.yaml`, `values-prd.yaml`) | YAML puro com uma base e sobreposições (*overlays*) por ambiente |
| Ponto forte | Um **chart padrão** reutilizado por todos os serviços, com versionamento | Simplicidade e leitura direta, sem linguagem de template |
| Quando usar | Plataformas com muitos serviços parecidos | Poucos serviços ou diferenças pequenas entre ambientes |

Uma estratégia eficiente: o time de plataforma mantém um **chart Helm padrão de microsserviço**, com todas as boas práticas da seção 7.2 já embutidas, e cada serviço só informa os valores:

```yaml
# servicos/pedidos/values-prd.yaml
imagem:
  repositorio: acrlojaprd.azurecr.io/pedidos
  digest: sha256:...
replicas: 3
recursos:
  requests: { cpu: 500m, memory: 512Mi }
  limits: { memory: 1Gi }
autoscaling: { min: 3, max: 15, cpuAlvo: 70 }
workloadIdentity:
  clientId: 00000000-0000-0000-0000-000000000000
config:
  ServiceBus__Namespace: sb-loja-prd.servicebus.windows.net
```

```bash
helm upgrade --install pedidos oci://acrlojaprd.azurecr.io/charts/microsservico \
  --version 2.3.0 -n pedidos -f servicos/pedidos/values-prd.yaml
```

Charts também podem ser publicados no ACR como artefatos OCI, junto das imagens.

### Laboratório

- [ ] Crie os namespaces dos seis serviços com Pod Security `restricted` e cotas.
- [ ] Implante o `pedidos-api` com o Deployment da seção 7.2 e confirme, com `kubectl get pods -o wide`, que as réplicas ficaram em zonas diferentes.
- [ ] Crie o PDB e simule a manutenção de um nó com `kubectl drain`, observando que o serviço continua no ar.
- [ ] Transforme o manifesto em um chart Helm padrão e implante dois serviços diferentes com ele.

## 8. Entrada de tráfego, APIs e TLS

O tráfego externo passa por até três camadas, cada uma com uma função: a **borda** (Front Door) protege e acelera, a **API Management** governa o contrato das APIs e o **gateway do cluster** roteia para o serviço certo. Para APIs internas ou sistemas menores, as duas primeiras camadas são opcionais.

### 8.1 O fim do Ingress-NGINX e a Gateway API

> **Mudança importante:** o projeto comunitário Ingress-NGINX foi aposentado em março de 2026, e o suporte da Microsoft ao NGINX gerenciado do *application routing add-on* (apenas patches críticos de segurança) terminou em 30 de novembro de 2026 ([guia de migração](https://learn.microsoft.com/azure/aks/app-routing-nginx-to-gateway-api-migration), [aviso de aposentadoria](https://learn.microsoft.com/azure/aks/app-routing-migration)). Clusters novos devem usar a **Gateway API**, e clusters existentes com NGINX devem migrar.

A **Gateway API** é a sucessora do objeto `Ingress` no Kubernetes. Ela separa responsabilidades: o time de plataforma define o `Gateway` (endereço, portas, certificados) e cada time de serviço define as suas `HTTPRoute`.

| Opção de entrada no AKS | API | Quando usar |
| --- | --- | --- |
| **Application routing com Gateway API** (baseado em Istio, sem malha de serviços) | Gateway API | Padrão gerenciado dentro do cluster; é o padrão dos clusters AKS Automatic a partir da versão 1.36 |
| **Application Gateway for Containers (AGC)** | Gateway API e Ingress | Balanceador gerenciado fora do cluster, com WAF, alta escala e divisão de tráfego |
| **Istio service mesh add-on** com ingress gateway | Gateway API e APIs do Istio | Quando você já usa a malha de serviços (capítulo 9) |

O application routing com Gateway API e a instalação gerenciada da Gateway API estão disponíveis de forma geral desde a versão de 28 de abril de 2026 do AKS ([blog do AKS](https://blog.aks.azure.com/authors/jaiveer-katariya)).

### 8.2 Habilitando o application routing com Gateway API

```bash
az aks update -g $RG -n $CLUSTER --enable-gateway-api --enable-app-routing-istio
kubectl get gatewayclass          # confira o nome da classe criada pelo add-on
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: loja
  namespace: gateway
spec:
  gatewayClassName: approuting-istio      # use o nome listado pelo comando acima
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "api.loja.exemplo.com.br"
      tls:
        certificateRefs: [{ name: loja-tls }]
      allowedRoutes:
        namespaces: { from: Selector, selector: { matchLabels: { loja/rotas: "true" } } }
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: pedidos
  namespace: pedidos
spec:
  parentRefs: [{ name: loja, namespace: gateway }]
  hostnames: ["api.loja.exemplo.com.br"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /pedidos } }]
      backendRefs: [{ name: pedidos-api, port: 80 }]
```

A divisão de tráfego para *canary* vira apenas pesos na rota:

```yaml
      backendRefs:
        - { name: pedidos-api, port: 80, weight: 90 }
        - { name: pedidos-api-canary, port: 80, weight: 10 }
```

### 8.3 Application Gateway for Containers

O **AGC** é um balanceador de camada 7 gerenciado pelo Azure, configurado a partir do cluster por um controlador (ALB Controller). Use-o quando quiser tirar a entrada de dentro do cluster, precisar de WAF integrado ou de grande escala. Ele usa a classe `azure-alb-external` na Gateway API e uma sub-rede delegada na VNet (capítulo 4).

### 8.4 API Management

A **Azure API Management (APIM)** é a porta de entrada das APIs para parceiros, aplicativos e outros times. Ela fica na frente do gateway do cluster e adiciona:

- autenticação com tokens do Entra ID ou de outro provedor OIDC (`validate-jwt`);
- limites de uso e cotas por cliente (`rate-limit-by-key`, `quota-by-key`);
- versões e revisões das APIs, com o contrato OpenAPI publicado;
- transformações, cache de respostas e portal do desenvolvedor.

```xml
<policies>
  <inbound>
    <base />
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
      <audiences><audience>api://loja-pedidos</audience></audiences>
    </validate-jwt>
    <rate-limit-by-key calls="100" renewal-period="60"
                       counter-key="@(context.Subscription?.Key ?? context.Request.IpAddress)" />
    <set-backend-service base-url="https://api-interna.loja.exemplo.com.br/pedidos" />
  </inbound>
</policies>
```

Para uma conexão privada até o cluster, use uma camada da APIM com integração à VNet e um gateway interno (privado) no AKS.

### 8.5 Front Door, DNS e certificados

- **Azure Front Door com WAF:** TLS na borda, regras gerenciadas contra ataques comuns (OWASP), proteção contra bots e roteamento entre regiões (capítulo 16). Conecte-o à origem por **Private Link** para que a origem não tenha IP público exposto.
- **DNS:** gerencie as zonas no Azure DNS; o application routing add-on pode criar os registros automaticamente.
- **Certificados:** guarde-os no **Key Vault**. O add-on consegue usar certificados do Key Vault; outra opção comum é o **cert-manager** com Let's Encrypt.

### 8.6 Tráfego entre serviços

Dentro do cluster, os serviços conversam diretamente pelos `Service` do Kubernetes, sem passar pela APIM ou pelo gateway. Proteja esse tráfego com **network policies** (capítulo 15) e, se precisar de mTLS e de políticas finas por serviço, com uma malha de serviços (capítulo 9).

### Laboratório

- [ ] Habilite a Gateway API e o application routing com Istio, crie o `Gateway` e uma `HTTPRoute` para cada serviço público.
- [ ] Faça um canary de 10% para uma nova versão de `catalogo` usando pesos na `HTTPRoute`.
- [ ] Se você tiver um cluster com NGINX, siga o guia oficial de migração e converta um `Ingress` para `HTTPRoute`.
- [ ] Crie uma APIM de desenvolvimento, importe a especificação OpenAPI de `pedidos` e aplique a política de limite de uso.

## 9. Comunicação e mensageria

A mensageria é o que permite que os serviços da Loja funcionem mesmo quando um deles está lento ou fora do ar: o pedido é aceito, o evento fica guardado no broker e o serviço de pagamentos o processa quando voltar.

### 9.1 Escolhendo o serviço de mensagens

| Serviço | Modelo | Use para |
| --- | --- | --- |
| **Azure Service Bus** | Filas e tópicos com assinaturas, entrega garantida, transações, sessões, *dead-letter* | Comandos e eventos de negócio entre microsserviços (a escolha padrão) |
| **Azure Event Hubs** | Log de eventos particionado, alta vazão, retenção e replay, compatível com Kafka | Telemetria, *clickstream*, ingestão de grandes volumes, *event sourcing* em escala |
| **Azure Event Grid** | Roteamento de eventos reativos, push via HTTP | Eventos de recursos do Azure (um arquivo novo no Storage) e integrações simples |

### 9.2 Modelando com Service Bus

- **Tópico** `pedidos-eventos` recebe todos os eventos do serviço `pedidos`; cada serviço interessado tem sua **assinatura** (`estoque`, `pagamentos`, `notificacoes`), com filtros por tipo de evento.
- **Fila** para comandos ponto a ponto, como `pagamentos-comandos`.
- **Sessões** garantem a ordem das mensagens de um mesmo pedido (sessão = id do pedido).
- **Detecção de duplicatas** descarta mensagens com o mesmo `MessageId` dentro de uma janela.
- **Dead-letter queue (DLQ)** recebe mensagens que falharam várias vezes ou expiraram; monitore-a e trate-a, nunca a ignore.

Publicando um evento em .NET, autenticado com Workload Identity (sem connection string):

```csharp
using Azure.Identity;
using Azure.Messaging.ServiceBus;

await using var client = new ServiceBusClient(
    "sb-loja-dev.servicebus.windows.net", new DefaultAzureCredential());
await using var sender = client.CreateSender("pedidos-eventos");

var mensagem = new ServiceBusMessage(BinaryData.FromObjectAsJson(evento))
{
    MessageId = evento.Id,                 // base para a detecção de duplicatas
    Subject = "PedidoCriado",              // usado nos filtros das assinaturas
    SessionId = evento.PedidoId,
    ContentType = "application/cloudevents+json"
};
await sender.SendMessageAsync(mensagem);
```

Consumindo com processamento idempotente:

```csharp
var processor = client.CreateSessionProcessor("pedidos-eventos", "estoque",
    new ServiceBusSessionProcessorOptions { AutoCompleteMessages = false, MaxConcurrentSessions = 16 });

processor.ProcessMessageAsync += async args =>
{
    if (await jaProcessado(args.Message.MessageId)) { await args.CompleteMessageAsync(args.Message); return; }
    await reservarEstoque(args.Message.Body.ToObjectFromJson<PedidoCriado>());
    await registrarProcessado(args.Message.MessageId);
    await args.CompleteMessageAsync(args.Message);
};
processor.ProcessErrorAsync += args => { logger.LogError(args.Exception, "Erro no Service Bus"); return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

### 9.3 Dapr: blocos prontos para microsserviços

O **Dapr** (*Distributed Application Runtime*) roda como *sidecar* ao lado de cada serviço e oferece APIs padronizadas para pub/sub, estado, invocação entre serviços, segredos e workflows. O código chama o Dapr por HTTP ou gRPC, e a tecnologia por trás (Service Bus, Redis, Cosmos DB) é configuração. No AKS, ele é instalado como **extensão gerenciada**:

```bash
az k8s-extension create --cluster-type managedClusters \
  --cluster-name $CLUSTER --resource-group $RG \
  --name dapr --extension-type Microsoft.Dapr
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eventos
  namespace: pedidos
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
    - name: namespaceName
      value: sb-loja-dev.servicebus.windows.net
    - name: azureClientId                 # identidade da Workload Identity
      value: 00000000-0000-0000-0000-000000000000
```

Com isso, publicar um evento é um `POST http://localhost:3500/v1.0/publish/eventos/pedidos-eventos`, em qualquer linguagem. O **Dapr Workflow** é uma boa opção para orquestrar a saga do capítulo 3.

### 9.4 Service mesh com o add-on do Istio

Uma **malha de serviços** coloca um proxy (Envoy) ao lado de cada Pod e controla todo o tráfego entre serviços, sem mudar o código:

- **mTLS** automático entre serviços, com identidade por serviço;
- **políticas de autorização** (“só `pedidos` pode chamar `estoque`”);
- **retries, timeouts e circuit breakers** configurados de forma centralizada;
- **divisão de tráfego** fina entre versões e telemetria por requisição.

```bash
az aks mesh enable -g $RG -n $CLUSTER
az aks show -g $RG -n $CLUSTER --query serviceMeshProfile.istio.revisions   # revisão instalada
kubectl label namespace pedidos istio.io/rev=asm-1-xx     # use a revisão do comando anterior
```

A malha adiciona latência, consumo de recursos e complexidade operacional. Adote-a quando os requisitos (mTLS obrigatório, políticas entre serviços, tráfego avançado) justificarem; caso contrário, network policies e bibliotecas de resiliência no código costumam bastar.

### 9.5 Resiliência nas chamadas síncronas

Toda chamada entre serviços precisa de **timeout**, **retry com backoff exponencial e jitter** (só para operações idempotentes) e **circuit breaker**. Em .NET, o pacote `Microsoft.Extensions.Http.Resilience` aplica um conjunto padrão com uma linha:

```csharp
builder.Services.AddHttpClient<EstoqueClient>(c =>
        c.BaseAddress = new Uri("http://estoque-api.estoque.svc.cluster.local"))
    .AddStandardResilienceHandler();
```

Em Java, use o Resilience4j; em Node.js, bibliotecas como `cockatiel`. Não empilhe retries em várias camadas (código, Dapr e Istio ao mesmo tempo), ou uma falha se multiplica em uma tempestade de requisições.

### Laboratório

- [ ] Crie um namespace do Service Bus com o tópico `pedidos-eventos` e as assinaturas de `estoque`, `pagamentos` e `notificacoes`, com filtros por `Subject`.
- [ ] Implemente a publicação com outbox no serviço `pedidos` e o consumo idempotente em `estoque`.
- [ ] Force falhas no consumidor e observe as mensagens chegando à dead-letter queue.
- [ ] Instale a extensão do Dapr e troque a publicação direta por um componente de pub/sub.

## 10. Dados

A recomendação padrão é manter os bancos **fora do cluster**, em serviços gerenciados: backup, alta disponibilidade, patches e escala ficam com o Azure, e o cluster pode ser recriado a qualquer momento sem risco aos dados. Cada serviço tem o seu banco (capítulo 3), escolhido pelo formato dos dados e pelo padrão de acesso.

### 10.1 Escolhendo o banco de cada serviço

| Serviço | Banco | Por quê |
| --- | --- | --- |
| `catalogo` | **Azure Cosmos DB** (NoSQL) | Documentos flexíveis, leitura muito alta, distribuição global, latência baixa |
| `carrinho` | **Azure Cache for Redis** (ou Azure Managed Redis) | Dados temporários por sessão, acesso em milissegundos, expiração automática |
| `pedidos` | **Azure SQL Database** | Transações, integridade relacional, relatórios; time com experiência em SQL Server |
| `pagamentos` | **Azure Database for PostgreSQL Flexible Server** | Relacional, open source, consistência forte |
| `estoque` | **Azure Database for PostgreSQL Flexible Server** (servidor ou banco separado) | Relacional, controle de concorrência nas reservas |

Em ambientes de desenvolvimento, vários serviços podem compartilhar um mesmo servidor com **bancos separados** para economizar; em produção, separe servidores quando os requisitos de escala, isolamento ou janela de manutenção forem diferentes.

### 10.2 Conexão segura e sem senha

1. **Rede:** acesso público desabilitado e **Private Endpoint** na sub-rede `snet-private-endpoints`, com zona DNS privada vinculada à VNet.
2. **Identidade:** autenticação pelo **Entra ID** usando a identidade do serviço (Workload Identity), sem usuário e senha no código ou em segredos.
3. **Permissões mínimas:** a identidade de cada serviço só acessa o próprio banco.

```csharp
// Azure SQL com a identidade do Pod: "Authentication=Active Directory Default" usa o DefaultAzureCredential
var conexao = "Server=tcp:sql-loja-dev.database.windows.net;Database=pedidos;" +
              "Authentication=Active Directory Default;Encrypt=True;";
```

```java
// Cosmos DB em Java, sem chave de acesso
CosmosClient cliente = new CosmosClientBuilder()
    .endpoint("https://cosmos-loja-dev.documents.azure.com:443/")
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
```

### 10.3 Modelagem no Cosmos DB

A escolha da **chave de partição** define desempenho e custo. Ela deve ter muitos valores distintos, distribuir bem as leituras e escritas e estar presente nas consultas mais frequentes. Para o catálogo, `categoriaId` costuma ser ruim (poucas categorias concentram tráfego); `produtoId` distribui bem, e consultas por categoria podem ser atendidas por um índice de busca (Azure AI Search) alimentado por eventos. Use o modo **autoscale** de throughput ou o **serverless** para cargas irregulares.

### 10.4 Migrações de esquema

Cada serviço versiona as próprias migrações (EF Core Migrations, Flyway, Liquibase) no seu repositório. Execute-as como um **Job do Kubernetes** ou uma etapa do pipeline **antes** do deploy da nova versão, nunca na inicialização de cada réplica, que disputariam a mesma migração.

Como o deploy é gradual, as versões antiga e nova do serviço rodam ao mesmo tempo. Por isso as mudanças de esquema seguem o padrão **expandir e contrair**: primeiro adicione (coluna nova, opcional), implante o código que usa as duas formas, migre os dados e só depois remova o que ficou obsoleto.

### 10.5 Cache e consultas entre serviços

- **Cache-aside** com Redis para leituras frequentes de outro serviço (por exemplo, `carrinho` guardando preços do `catalogo` por alguns minutos).
- **Modelos de leitura** (CQRS) alimentados por eventos para consultas que juntam dados de vários serviços, como “meus pedidos com status de pagamento e entrega”.
- Nunca resolva uma consulta entre serviços lendo diretamente o banco do outro.

### 10.6 Backup e retenção

| Banco | Recurso | Verifique |
| --- | --- | --- |
| Azure SQL | Backups automáticos com restauração para um ponto no tempo; retenção de longo prazo opcional | Período de retenção e geo-redundância do backup |
| PostgreSQL Flexible | Backups automáticos e restauração para um ponto no tempo; alta disponibilidade com zona redundante | Retenção e réplica de leitura em outra região |
| Cosmos DB | Backup periódico ou contínuo (restauração para um ponto no tempo) | Modo de backup e regiões |
| Redis | Persistência e geo-replicação nos níveis mais altos | Se o carrinho pode ser perdido em um desastre |

Teste a restauração periodicamente: um backup que nunca foi restaurado é uma suposição.

### 10.7 E bancos dentro do cluster?

É possível rodar bancos no AKS com StatefulSets, discos gerenciados (Azure Disk CSI) e operadores como o CloudNativePG, e isso pode reduzir custo ou atender a requisitos específicos. Em troca, você assume backup, alta disponibilidade, upgrades e tuning. Para a maioria das equipes de produto, serviços gerenciados são a escolha mais segura.

### Laboratório

- [ ] Crie o Azure SQL, um PostgreSQL Flexible, uma conta do Cosmos DB e um Redis com Private Endpoints.
- [ ] Conecte `pedidos` ao Azure SQL e `catalogo` ao Cosmos DB usando apenas Workload Identity.
- [ ] Execute as migrações de `pedidos` como um Job antes do deploy e faça uma mudança no padrão expandir e contrair.
- [ ] Restaure o banco de `pedidos` para um ponto no tempo em um banco novo e valide os dados.

## 11. Identidade e segredos

O objetivo é ter **zero segredos de longa duração** no cluster: nenhuma connection string com senha, nenhuma chave de acesso em variável de ambiente. Os Pods provam quem são ao Entra ID e recebem tokens de curta duração; o que ainda for segredo (por exemplo, a chave de API de um parceiro) fica no Key Vault.

### 11.1 Como funciona a Workload Identity

1. O cluster expõe um **emissor OIDC** (`--enable-oidc-issuer`).
2. Cada serviço tem uma **ServiceAccount** do Kubernetes e uma **identidade gerenciada** no Azure.
3. Uma **credencial federada** diz ao Entra ID: “confie em tokens desta ServiceAccount, emitidos por este cluster, como se fossem desta identidade”.
4. O Pod recebe um token da ServiceAccount, e os SDKs do Azure (com `DefaultAzureCredential`) o trocam por um token do Entra ID.

### 11.2 Configurando para o serviço `pedidos`

```bash
export OIDC_ISSUER=$(az aks show -g $RG -n $CLUSTER --query oidcIssuerProfile.issuerUrl -o tsv)

az identity create -g $RG -n id-pedidos
export CLIENT_ID=$(az identity show -g $RG -n id-pedidos --query clientId -o tsv)
export PRINCIPAL_ID=$(az identity show -g $RG -n id-pedidos --query principalId -o tsv)

az identity federated-credential create -g $RG --identity-name id-pedidos \
  --name aks-pedidos --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:pedidos:pedidos-api \
  --audience api://AzureADTokenExchange

# permissões mínimas: enviar e receber no Service Bus
az role assignment create --assignee-object-id $PRINCIPAL_ID --assignee-principal-type ServicePrincipal \
  --role "Azure Service Bus Data Sender" \
  --scope $(az servicebus namespace show -g $RG -n sb-loja-dev --query id -o tsv)
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pedidos-api
  namespace: pedidos
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
```

O Deployment do capítulo 7 já usa essa ServiceAccount e o label `azure.workload.identity/use: "true"`, que faz o AKS injetar o token e as variáveis necessárias no Pod. No código, basta `new DefaultAzureCredential()` (.NET), `DefaultAzureCredentialBuilder` (Java) ou `DefaultAzureCredential` do `@azure/identity` (Node.js).

| Recurso | Papel mínimo para um serviço |
| --- | --- |
| Service Bus | Azure Service Bus Data Sender e/ou Data Receiver, no tópico ou fila |
| Key Vault | Key Vault Secrets User, no cofre do serviço |
| Storage | Storage Blob Data Reader ou Contributor, no container |
| Cosmos DB | Papel de dados do Cosmos DB (Built-in Data Contributor), na conta ou no container |
| Azure SQL / PostgreSQL | Usuário do banco criado a partir da identidade do Entra ID, com permissões SQL mínimas |

### 11.3 Segredos com Key Vault

Para o que ainda é segredo, use o **Azure Key Vault**, com um cofre por serviço (ou por domínio) e o modelo de permissões por **RBAC do Azure**. Há duas formas de entregar o segredo ao Pod:

| Forma | Como funciona | Quando usar |
| --- | --- | --- |
| SDK no código | A aplicação lê o segredo do Key Vault com `DefaultAzureCredential` | Preferida: sem cópia do segredo no cluster, rotação simples |
| **Secrets Store CSI Driver** (add-on do Key Vault) | O segredo é montado como arquivo no Pod (e, opcionalmente, sincronizado como `Secret` do Kubernetes) | Aplicações que só sabem ler arquivo ou variável de ambiente |

```bash
az aks enable-addons -g $RG -n $CLUSTER --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation
```

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: pagamentos-segredos
  namespace: pagamentos
spec:
  provider: azure
  parameters:
    clientID: "<CLIENT_ID da identidade de pagamentos>"   # autentica com Workload Identity
    keyvaultName: kv-loja-pagamentos-dev
    tenantId: "<TENANT_ID>"
    objects: |
      array:
        - |
          objectName: chave-api-gateway-pagamento
          objectType: secret
```

No Deployment, monte o volume `csi` com `driver: secrets-store.csi.k8s.io` e `volumeAttributes.secretProviderClass: pagamentos-segredos`; o segredo aparece como arquivo em `/mnt/segredos/chave-api-gateway-pagamento`.

### 11.4 Acesso de pessoas ao cluster

O capítulo 5 configurou o login pelo Entra ID e o Azure RBAC para Kubernetes. Complete com:

- **Contas locais desabilitadas** (`--disable-local-accounts`): ninguém usa o certificado de administrador.
- **Acesso de leitura por padrão** para desenvolvedores em produção; mudanças só pelo GitOps.
- **Acesso emergencial** (*break glass*) por um grupo elevado temporariamente com PIM, com auditoria.
- **Logs de auditoria do API server** enviados ao Log Analytics (capítulo 13).

### Laboratório

- [ ] Configure a Workload Identity de `pedidos` e publique uma mensagem no Service Bus sem nenhuma connection string.
- [ ] Remova o papel da identidade e confirme que o serviço passa a receber erro de autorização.
- [ ] Crie um Key Vault para `pagamentos`, guarde um segredo e leia-o pelo SDK e pelo CSI Driver.
- [ ] Procure no cluster qualquer `Secret` com credenciais de serviços do Azure e elimine-o.

## 12. Escalabilidade

No AKS a escala acontece em dois níveis que trabalham juntos: **Pods** (quantas réplicas de cada serviço) e **nós** (quantas VMs existem para hospedá-los). Quando o HPA ou o KEDA criam Pods que não cabem nos nós atuais, o cluster autoscaler ou o node auto-provisioning adicionam nós.

| Ferramenta | Escala | Com base em |
| --- | --- | --- |
| **HPA** (Horizontal Pod Autoscaler) | Número de Pods | CPU, memória ou métricas customizadas |
| **KEDA** | Número de Pods, inclusive até zero | Eventos: mensagens na fila, lag do Event Hubs, Prometheus, agenda (cron) |
| **VPA** (Vertical Pod Autoscaler) | `requests` de cada Pod | Consumo observado |
| **Cluster autoscaler** | Nós dentro de node pools definidos | Pods pendentes por falta de recursos |
| **Node auto-provisioning (NAP)** | Nós de tamanhos escolhidos automaticamente | Requisitos dos Pods pendentes (baseado em Karpenter) |

### 12.1 HPA para as APIs

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: catalogo-api, namespace: catalogo }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: catalogo-api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300     # evita oscilações
```

O HPA calcula a utilização em relação aos `requests`; sem eles, não funciona. Mantenha `minReplicas` em pelo menos 3 para serviços críticos, um por zona.

### 12.2 KEDA para os consumidores de eventos

Para `estoque`, `pagamentos` e `notificacoes`, a métrica certa não é CPU, e sim **quantas mensagens estão esperando**. O **KEDA** é um add-on gerenciado do AKS:

```bash
az aks update -g $RG -n $CLUSTER --enable-keda
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata: { name: servicebus-wi, namespace: notificacoes }
spec:
  podIdentity:
    provider: azure-workload               # usa a Workload Identity, sem connection string
    identityId: "<CLIENT_ID com papel Azure Service Bus Data Receiver>"
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: notificacoes-worker, namespace: notificacoes }
spec:
  scaleTargetRef: { name: notificacoes-worker }
  minReplicaCount: 0                       # escala a zero quando não há mensagens
  maxReplicaCount: 20
  triggers:
    - type: azure-servicebus
      authenticationRef: { name: servicebus-wi }
      metadata:
        namespace: sb-loja-dev
        topicName: pedidos-eventos
        subscriptionName: notificacoes
        messageCount: "50"                 # uma réplica para cada 50 mensagens pendentes
```

O KEDA também escala **Jobs** (`ScaledJob`) para processamentos em lote e usa gatilhos de agenda para aumentar a capacidade antes de picos conhecidos, como a Black Friday.

### 12.3 Escala de nós

**Cluster autoscaler:** já habilitado nos node pools do capítulo 5. Ajuste o perfil quando necessário, por exemplo para remover nós ociosos mais rápido em desenvolvimento:

```bash
az aks update -g $RG -n $CLUSTER \
  --cluster-autoscaler-profile scale-down-unneeded-time=5m,expander=least-waste
```

**Node auto-provisioning (NAP):** em vez de você escolher o tamanho de VM de cada pool, o AKS (com Karpenter) escolhe e cria o nó mais adequado para os Pods pendentes, e consolida nós subutilizados. O NAP está disponível de forma geral desde julho de 2025 ([anúncio](https://mc.merill.net/message/AZ498247)) e é o padrão do AKS Automatic. Ele exige rede Overlay com dataplane Cilium:

```bash
az aks update -g $RG -n $CLUSTER --node-provisioning-mode Auto
```

Com o NAP, você descreve restrições (famílias de VM permitidas, zonas, spot ou sob demanda, limites de CPU) em recursos `NodePool` e `AKSNodeClass`, em vez de criar node pools manualmente.

### 12.4 Dimensionando bem

- Defina `requests` a partir do consumo real (Container Insights ou recomendações do VPA em modo só recomendação), não por palpite.
- Faça **testes de carga** (Azure Load Testing, k6, JMeter) antes de cada evento de pico e para validar o comportamento do autoscaling.
- Lembre que a escala dos Pods depende de dependências externas: um banco no limite não fica mais rápido com mais réplicas da API, e pode piorar com mais conexões.

### Laboratório

- [ ] Crie o HPA do `catalogo` e gere carga com o k6 ou o Azure Load Testing, acompanhando `kubectl get hpa -w`.
- [ ] Habilite o KEDA e configure o `notificacoes-worker` para escalar a partir de zero com base nas mensagens da assinatura.
- [ ] Publique 5.000 mensagens e observe Pods e nós sendo criados e depois removidos.
- [ ] Em um cluster de teste, habilite o NAP e compare os tamanhos de VM escolhidos com os dos node pools manuais.

## 13. Observabilidade

Com seis serviços, uma requisição lenta pode estar em qualquer um deles, no banco, na fila ou na rede. Observabilidade é conseguir responder “o que está acontecendo e por quê” sem precisar implantar código novo para descobrir. Ela se apoia em três sinais — **métricas**, **logs** e **traces** — ligados entre si pelo id de rastreamento.

### 13.1 A pilha no Azure

| Sinal | Serviço | O que coleta |
| --- | --- | --- |
| Métricas | **Azure Monitor managed service for Prometheus** | Métricas do cluster, dos nós, do Kubernetes e das aplicações (formato Prometheus) |
| Dashboards | **Azure Managed Grafana** | Painéis prontos para Kubernetes e painéis dos serviços |
| Logs | **Container Insights** + Log Analytics | Logs dos contêineres (stdout), eventos do Kubernetes, inventário |
| Traces e APM | **Application Insights** com **OpenTelemetry** | Rastreamento distribuído, dependências, exceções, mapa da aplicação |
| Plano de controle | Configurações de diagnóstico do AKS | Logs de auditoria do API server, do autoscaler e de outros componentes |

```bash
az aks update -g $RG -n $CLUSTER \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id $AMW_ID \
  --grafana-resource-id $GRAFANA_ID

az aks enable-addons -g $RG -n $CLUSTER --addons monitoring \
  --workspace-resource-id $LOG_ANALYTICS_ID
```

### 13.2 Instrumentando os serviços com OpenTelemetry

O **OpenTelemetry** é o padrão aberto de instrumentação. Em .NET, a distribuição do Azure Monitor configura traces, métricas e logs com poucas linhas:

```csharp
builder.Services.AddOpenTelemetry()
    .UseAzureMonitor()                         // envia ao Application Insights
    .ConfigureResource(r => r.AddService("pedidos-api"));
```

A connection string do Application Insights não é um segredo crítico, mas pode ir em um ConfigMap (`APPLICATIONINSIGHTS_CONNECTION_STRING`). Em Java, use o agente do Application Insights (`-javaagent:applicationinsights-agent.jar`), que instrumenta Spring, JDBC e HTTP sem mudança de código; em Node.js, o pacote `@azure/monitor-opentelemetry`.

O contexto do trace (cabeçalho `traceparent`, do padrão W3C) atravessa chamadas HTTP e mensagens do Service Bus. Assim, uma compra aparece como **um único trace** passando por `pedidos`, Service Bus, `estoque`, `pagamentos` e `notificacoes`.

### 13.3 Logs estruturados

- Escreva logs em **JSON** na saída padrão, com campos fixos: `timestamp`, `level`, `servico`, `trace_id`, `pedido_id`, `mensagem`.
- Inclua o `trace_id` em todo log, para ir do log ao trace e vice-versa.
- Nunca registre dados pessoais completos, tokens ou senhas.
- Controle o volume: logs de depuração desligados em produção, e tabelas de log no plano **Basic** do Log Analytics quando a consulta frequente não for necessária, o que reduz bastante o custo.

```kusto
// Erros por serviço nas últimas 2 horas (Container Insights)
ContainerLogV2
| where TimeGenerated > ago(2h)
| where LogLevel in ("error", "critical")
| summarize erros = count() by PodNamespace, bin(TimeGenerated, 5m)
| render timechart
```

### 13.4 Sinais dourados e SLOs

Para cada serviço, acompanhe os quatro **sinais dourados**: **latência**, **tráfego**, **erros** e **saturação**. Para consumidores de eventos, acrescente o **tamanho da fila** e a **idade da mensagem mais antiga**.

Defina **SLOs** (objetivos de nível de serviço) com o negócio, por exemplo: “99,9% das requisições de `POST /pedidos` respondidas com sucesso em menos de 500 ms, medidos em 30 dias”. O SLO define o **orçamento de erro**: enquanto ele não for consumido, o time pode implantar com frequência; quando estiver no limite, a prioridade passa a ser estabilidade.

```promql
# Taxa de erros 5xx do serviço pedidos (métrica de servidor HTTP do OpenTelemetry)
sum(rate(http_server_request_duration_seconds_count{service_name="pedidos-api", http_response_status_code=~"5.."}[5m]))
/
sum(rate(http_server_request_duration_seconds_count{service_name="pedidos-api"}[5m]))
```

Os nomes exatos das métricas dependem da biblioteca e da versão da instrumentação; confira-os no Grafana antes de criar alertas.

### 13.5 Alertas que valem a pena

| Alerta | Fonte |
| --- | --- |
| Consumo rápido do orçamento de erro de um SLO | Prometheus (regras de alerta do Azure Monitor) |
| Pods em `CrashLoopBackOff` ou reinicializações frequentes | Regras recomendadas do Container Insights / Prometheus |
| Nós `NotReady` ou com pressão de memória ou disco | Regras recomendadas do Prometheus |
| Mensagens na dead-letter queue | Métricas do Service Bus |
| Fila crescendo sem parar | Métricas do Service Bus ou do KEDA |
| Certificados perto de expirar | Key Vault (eventos) |

Comece pelas **regras de alerta recomendadas** que o Azure Monitor oferece para AKS e acrescente as de negócio. Cada alerta precisa de um **dono** e de um **runbook** com os primeiros passos de investigação.

### Laboratório

- [ ] Habilite o Managed Prometheus, o Managed Grafana e o Container Insights no cluster.
- [ ] Instrumente `pedidos` e `estoque` com OpenTelemetry e encontre, no Application Insights, um trace de compra completo.
- [ ] Escreva uma consulta KQL de erros por serviço e uma consulta PromQL de latência p95.
- [ ] Defina um SLO para `POST /pedidos` e crie um alerta de consumo do orçamento de erro.

## 14. CI/CD e GitOps

Cada microsserviço tem seu próprio pipeline e é implantado de forma independente; essa é a principal vantagem da arquitetura. O modelo recomendado separa duas responsabilidades:

- **CI (integração contínua):** no repositório do serviço, testa, constrói, analisa e publica a imagem no ACR.
- **CD com GitOps:** um repositório de configuração descreve o estado desejado de cada ambiente; um agente no cluster (**Flux** ou **Argo CD**) aplica continuamente esse estado. Ninguém roda `kubectl apply` em produção.

### 14.1 O fluxo completo

1. O desenvolvedor abre um PR no repositório de `pedidos`; o CI roda testes unitários, de contrato e análise estática.
2. No merge, o CI constrói a imagem, gera o SBOM, faz a varredura de vulnerabilidades, assina e publica no ACR com a tag do commit.
3. O CI abre automaticamente um PR no **repositório de configuração**, atualizando o digest da imagem no ambiente `dev`.
4. O Flux aplica a mudança no cluster de `dev`; testes de integração rodam contra o ambiente.
5. A promoção para `hml` e `prd` é um PR no repositório de configuração, com aprovação obrigatória para produção.
6. Rollback é um `git revert` no repositório de configuração.

### 14.2 CI com GitHub Actions e OIDC

O GitHub Actions se autentica no Azure por **federação OIDC**, sem segredo armazenado. Crie uma identidade gerenciada (ou um app registration) com uma credencial federada para o repositório e dê a ela apenas o papel `AcrPush` no registro.

```yaml
# .github/workflows/ci.yaml (repositório do serviço pedidos)
name: ci-pedidos
on:
  pull_request:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: "10.0.x" }
      - run: dotnet test --configuration Release

      - uses: azure/login@v2
        if: github.ref == 'refs/heads/main'
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Build, scan e push
        if: github.ref == 'refs/heads/main'
        run: |
          az acr login --name acrlojadev
          IMG=acrlojadev.azurecr.io/pedidos:${GITHUB_SHA::7}
          docker build -t $IMG .
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \
            --exit-code 1 --severity CRITICAL $IMG
          docker push $IMG
```

No Azure DevOps, o equivalente usa uma **service connection com Workload Identity Federation** e as tarefas `AzureCLI@2` e `Docker@2`.

### 14.3 GitOps com Flux

O Flux é instalado como extensão gerenciada do AKS e aponta para o repositório de configuração:

```bash
az k8s-configuration flux create -g $RG -c $CLUSTER -t managedClusters \
  --name loja --namespace flux-system --scope cluster \
  --url https://github.com/minha-org/loja-config --branch main \
  --kustomization name=infra path=./infra/dev prune=true \
  --kustomization name=apps path=./apps/dev prune=true dependsOn=["infra"]
```

```text
loja-config/
├── infra/
│   ├── base/            # namespaces, Gateway, políticas, KEDA, SecretProviderClass
│   ├── dev/
│   └── prd/
└── apps/
    ├── base/            # um HelmRelease (ou Kustomization) por serviço
    │   ├── pedidos/
    │   └── catalogo/
    ├── dev/             # sobreposições: digest da imagem, réplicas, configuração
    └── prd/
```

O **Argo CD** é a alternativa mais comum, com uma interface web forte para visualizar o estado das aplicações. Os dois seguem o mesmo princípio: o Git é a fonte da verdade, e qualquer mudança manual no cluster é detectada e revertida.

### 14.4 Estratégias de implantação

| Estratégia | Como funciona no AKS | Quando usar |
| --- | --- | --- |
| Rolling update | Padrão do Deployment, com `maxUnavailable: 0` | A maioria das mudanças |
| Canary | Pesos na `HTTPRoute` (capítulo 8), automatizado por Flagger ou Argo Rollouts com análise de métricas | Mudanças de risco em serviços com muito tráfego |
| Blue-green | Duas versões completas; a rota troca de uma vez | Quando duas versões não podem receber tráfego ao mesmo tempo |
| Feature flags | Código novo implantado desligado e ativado por configuração (Azure App Configuration) | Separar implantação de lançamento |

Com **canary automatizado**, a ferramenta aumenta o tráfego aos poucos (5%, 25%, 50%, 100%) e consulta o Prometheus a cada passo: se a taxa de erros ou a latência piorarem, volta automaticamente para a versão anterior.

### 14.5 Ambientes

- **Clusters separados** para produção e não produção, em assinaturas diferentes.
- **Mesma configuração base** em todos os ambientes; só mudam as sobreposições (réplicas, tamanhos, endereços).
- **Mesmo artefato** (digest da imagem) promovido entre ambientes, nunca reconstruído.
- **Ambientes efêmeros** por PR, em um namespace próprio, para testar mudanças grandes antes do merge.

### Laboratório

- [ ] Configure a federação OIDC entre o GitHub e o Azure e o workflow de CI do serviço `pedidos`.
- [ ] Crie o repositório de configuração com a estrutura acima e instale o Flux no cluster de desenvolvimento.
- [ ] Faça o CI abrir um PR no repositório de configuração com o novo digest e veja o Flux aplicar a mudança.
- [ ] Altere um Deployment manualmente com `kubectl` e observe o Flux revertê-lo; depois faça um rollback com `git revert`.

## 15. Segurança, governança e custos

A segurança de uma plataforma de microsserviços é feita em camadas: se uma falhar, a próxima limita o estrago. Os capítulos anteriores já construíram várias delas (identidade sem segredos, rede privada, imagens não root); este capítulo completa o conjunto e o transforma em regras obrigatórias.

### 15.1 Camadas de defesa

| Camada | Controles | Capítulo |
| --- | --- | --- |
| Identidade | Entra ID, Azure RBAC para Kubernetes, contas locais desabilitadas, Workload Identity, PIM | 4, 5, 11 |
| Rede | API server privado ou restrito, VNet privada, Private Endpoints, **network policies**, WAF na borda, saída pelo firewall | 4, 8, 15 |
| Imagens | Registro privado, varredura, assinatura, somente registros e imagens confiáveis | 6 |
| Workloads | Pod Security `restricted`, não root, sistema de arquivos somente leitura, sem capacidades extras | 7 |
| Dados | Criptografia em repouso e em trânsito, sem senhas, Key Vault, backups | 10, 11 |
| Detecção | Defender for Containers, logs de auditoria, alertas | 13, 15 |
| Governança | Azure Policy no cluster e na assinatura | 15 |

### 15.2 Network policies: zero confiança dentro do cluster

Por padrão, qualquer Pod pode falar com qualquer outro. Comece negando tudo em cada namespace e libere apenas o necessário:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: negar-tudo, namespace: estoque }
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: permitir-pedidos-e-gateway, namespace: estoque }
spec:
  podSelector:
    matchLabels: { app.kubernetes.io/name: estoque-api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: pedidos }
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: gateway }
      ports: [{ port: 8080, protocol: TCP }]
```

Com o dataplane Cilium (capítulo 4), as políticas são aplicadas com eBPF. Políticas de **saída** (`Egress`) também valem a pena: restrinja os Pods aos destinos de que precisam e lembre de liberar o DNS (CoreDNS na porta 53).

### 15.3 Azure Policy no cluster

O add-on do **Azure Policy** (baseado no Gatekeeper) aplica regras a todos os objetos do cluster e reporta a conformidade no portal:

```bash
az aks enable-addons -g $RG -n $CLUSTER --addons azure-policy
```

Atribua as iniciativas internas, como o **baseline** e o **restricted** de segurança de Pods, e políticas como:

- somente imagens de registros permitidos (`acrlojaprd.azurecr.io`);
- contêineres devem ter limites de memória definidos;
- sem contêineres privilegiados nem acesso à rede do host;
- serviços do tipo `LoadBalancer` apenas internos.

Comece com efeito **audit** para medir o impacto, corrija o que estiver fora e depois mude para **deny**. No AKS Automatic, os *deployment safeguards* aplicam um conjunto dessas boas práticas por padrão.

### 15.4 Microsoft Defender for Containers

O **Defender for Containers** junta três capacidades: avaliação de vulnerabilidades das imagens no ACR e em execução, recomendações de configuração do cluster e **detecção de ameaças em tempo de execução** (processos suspeitos, mineradores de criptomoeda, acesso a credenciais). Habilite-o na assinatura de produção e encaminhe os alertas para o time de segurança (por exemplo, pelo Microsoft Sentinel).

### 15.5 Lista rápida de endurecimento

- [ ] Contas locais desabilitadas e login só pelo Entra ID
- [ ] API server privado ou com faixas de IP autorizadas
- [ ] Node pools com Azure Linux ou Ubuntu atualizados automaticamente
- [ ] Pod Security `restricted` em todos os namespaces de aplicação
- [ ] Network policies de negação padrão
- [ ] Somente imagens assinadas e do registro corporativo
- [ ] Nenhum segredo de serviços do Azure no cluster
- [ ] Logs de auditoria do API server no Log Analytics
- [ ] Defender for Containers e Azure Policy habilitados

### 15.6 Custos

| Onde o custo aparece | Como reduzir |
| --- | --- |
| Nós ociosos | Autoscaling (capítulo 12), NAP para escolher o tamanho certo, `requests` realistas |
| Ambientes de desenvolvimento | `az aks stop` fora do expediente; clusters menores; spot |
| Workloads tolerantes a interrupção | Node pools spot |
| Capacidade estável de produção | Instâncias reservadas ou Savings Plans |
| Logs | Plano Basic do Log Analytics para logs de alto volume, retenção adequada, filtros na coleta |
| Bancos e cache | Tamanhos revisados, autoscale ou serverless no Cosmos DB |
| Tráfego de saída | Tudo na mesma região; Private Endpoints em vez de tráfego público |

Use a **análise de custos do AKS** para ver o custo por namespace e cobrar cada time pelo que consome (*showback*). Isso costuma ser o incentivo mais eficaz para dimensionar bem os serviços.

### Laboratório

- [ ] Aplique a política de negação padrão em `estoque` e libere apenas `pedidos` e o gateway; teste com um Pod temporário em outro namespace.
- [ ] Habilite o Azure Policy e atribua a iniciativa de segurança de Pods em modo audit; corrija as violações encontradas.
- [ ] Habilite o Defender for Containers e revise as recomendações do cluster.
- [ ] Habilite a análise de custos do AKS e identifique os três namespaces mais caros.

## 16. Resiliência e operação

Resiliência é planejar o que acontece quando algo falha: um Pod, um nó, uma zona, uma região inteira ou uma dependência externa. Comece definindo, com o negócio, dois números para cada serviço: **RTO** (quanto tempo ele pode ficar fora do ar) e **RPO** (quantos dados podem ser perdidos). Eles definem quanto vale a pena investir.

### 16.1 Falhas e proteções

| Falha | Proteção | Capítulo |
| --- | --- | --- |
| Um Pod trava | Probes e réplicas múltiplas | 7 |
| Um nó cai ou entra em manutenção | Réplicas em nós diferentes, PodDisruptionBudget, autoscaling | 7, 12 |
| Uma zona de disponibilidade cai | Nós em três zonas, `topologySpreadConstraints`, bancos com redundância de zona | 5, 7, 10 |
| Uma dependência fica lenta | Timeouts, retries, circuit breaker, mensageria assíncrona | 9 |
| Uma região inteira cai | Segunda região com Front Door e réplicas dos dados | 16 |
| Um erro humano apaga recursos | GitOps (recriação a partir do Git), backups, bloqueios de recursos | 14, 16 |

### 16.2 Multi-região

| Estratégia | Como funciona | RTO típico | Custo |
| --- | --- | --- | --- |
| Backup e restauração | Infraestrutura recriada por código em outra região; dados restaurados de backups geo-redundantes | Horas | Baixo |
| Ativo-passivo (*warm standby*) | Cluster menor já rodando na região secundária; bancos com réplica; Front Door troca a origem | Minutos | Médio |
| Ativo-ativo | As duas regiões atendem tráfego; dados com escrita em múltiplas regiões (por exemplo, Cosmos DB) | Próximo de zero | Alto e complexo |

Como cluster e configuração são código (Terraform e GitOps), recriar o AKS em outra região é relativamente rápido; a parte difícil são os **dados**. Planeje a replicação de cada banco e a ordem de failover das dependências.

### 16.3 Backup do cluster

Com GitOps e bancos gerenciados, quase todo o estado está fora do cluster. Para o que estiver dentro (volumes persistentes, recursos criados fora do Git), use o **Azure Backup para AKS** (baseado em Velero), que faz backup de recursos do Kubernetes e de discos, com restauração para outro cluster. Proteja também os recursos críticos do Azure com **bloqueios** (`CanNotDelete`).

### 16.4 Upgrades sem medo

- Atualize **não produção primeiro**, com o mesmo procedimento, e espere alguns dias antes de produção.
- Use canais de upgrade automático e janelas de manutenção (capítulo 5); acompanhe o calendário de versões para não ficar fora do suporte.
- Para mudanças grandes (versão menor com mudanças incompatíveis, troca de rede), considere **clusters azul-verde**: crie um cluster novo, aplique a configuração pelo GitOps, mova o tráfego aos poucos pelo Front Door ou pelo gateway e desligue o antigo.
- Teste os PDBs: um PDB que exige 100% de disponibilidade trava o upgrade dos nós.

### 16.5 Engenharia do caos

O **Azure Chaos Studio** injeta falhas controladas (desligar nós, matar Pods, adicionar latência na rede, derrubar uma zona) para provar que as proteções funcionam. Comece em homologação, com hipóteses claras: “se um nó cair, o p95 de `POST /pedidos` continua abaixo de 500 ms”.

### 16.6 Troubleshooting

Um roteiro, do mais comum para o mais raro:

| Sintoma | Comandos e verificações |
| --- | --- |
| Pod em `Pending` | `kubectl describe pod`: falta de recursos (veja o autoscaler), *taints*, PVC não vinculado, cota do namespace |
| `ImagePullBackOff` | Nome e tag da imagem; `az aks check-acr -g $RG -n $CLUSTER --acr <acr>.azurecr.io`; acesso de rede ao ACR privado |
| `CrashLoopBackOff` | `kubectl logs --previous`; configuração ausente; liveness probe agressiva demais |
| Pod reiniciando por `OOMKilled` | `kubectl describe pod`; aumente `limits.memory` ou investigue vazamento de memória |
| Erro 401/403 ao acessar o Azure | Label da Workload Identity no Pod, anotação da ServiceAccount, credencial federada (subject exato) e papéis atribuídos |
| Serviço não responde dentro do cluster | `kubectl get endpointslices`; seletor do Service; network policies; DNS (`nslookup` de um Pod temporário) |
| Rota externa não funciona | Status do `Gateway` e da `HTTPRoute` (`kubectl describe`), certificado, DNS público, regras do WAF |
| Latência alta | Traces no Application Insights; throttling de CPU; conexões do banco; filas crescendo |
| Nó `NotReady` | `kubectl describe node`; pressão de disco ou memória; painel **Diagnosticar e solucionar problemas** do AKS no portal |

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl top pods -A --sort-by=memory
kubectl debug -it <pod> --image=mcr.microsoft.com/azurelinux/base/core:3.0 --target=<container>
az aks command invoke -g $RG -n $CLUSTER --command "kubectl get pods -A"   # clusters privados
```

Mantenha um **runbook** por alerta e faça uma revisão sem culpados (*postmortem*) depois de cada incidente relevante, com ações para evitar a repetição.

### Laboratório

- [ ] Defina RTO e RPO para os seis serviços da Loja e escolha a estratégia de multi-região adequada.
- [ ] Configure o Azure Backup para AKS e restaure um namespace em outro cluster.
- [ ] Rode um experimento no Chaos Studio que desligue nós de uma zona e observe o comportamento dos serviços.
- [ ] Provoque três problemas da tabela 16.6 de propósito e resolva-os usando apenas o roteiro.

## 17. Projeto final, checklist de produção e trilha de estudos

O projeto final é a Loja Online completa: seis microsserviços implantados de forma independente em um AKS seguro, com mensageria, observabilidade e GitOps. Ao terminar, você terá um portfólio que cobre o que se espera de um engenheiro de plataforma ou de um desenvolvedor sênior de microsserviços.

### 17.1 Entregas

| # | Entrega | Critério de pronto | Capítulos |
| --- | --- | --- | --- |
| 1 | Design | Mapa de contextos, contratos OpenAPI e de eventos, ADRs das decisões principais | 2, 3 |
| 2 | Fundação | Rede, cluster, ACR, Key Vault e bancos criados só por Terraform ou Bicep | 4, 5 |
| 3 | Serviços | Seis serviços em ao menos duas linguagens, com imagens multi-stage não root e assinadas | 6 |
| 4 | Implantação | Chart padrão com probes, recursos, PDB, espalhamento por zonas e segurança `restricted` | 7 |
| 5 | Entrada | Gateway API com TLS, rotas por serviço, APIM com autenticação e limite de uso | 8 |
| 6 | Saga de compra | Fluxo de pedido com Service Bus, outbox, consumidores idempotentes e compensação testada | 3, 9 |
| 7 | Dados | Banco por serviço com Private Endpoint e autenticação sem senha; migrações como Job | 10 |
| 8 | Identidade | Workload Identity em todos os serviços; zero segredos de serviços do Azure no cluster | 11 |
| 9 | Escala | HPA nas APIs, KEDA nos consumidores (escala a zero), teste de carga documentado | 12 |
| 10 | Observabilidade | Trace de compra de ponta a ponta, dashboards, SLO de `POST /pedidos` com alerta | 13 |
| 11 | Entrega | CI com OIDC por serviço, GitOps com Flux, promoção por PR, canary em um serviço | 14 |
| 12 | Segurança e operação | Network policies, Azure Policy em modo deny, Defender, backup restaurado, experimento de caos e runbooks | 15, 16 |

### 17.2 Checklist de produção

**Cluster**

- [ ] Tier Standard (com SLA), nós em três zonas, pools de sistema e de usuário separados
- [ ] Versão do Kubernetes suportada, canais de upgrade e janela de manutenção configurados
- [ ] Login só pelo Entra ID, contas locais desabilitadas, API server privado ou restrito
- [ ] Rede Overlay com Cilium e faixas de IP sem sobreposição

**Serviços**

- [ ] Pelo menos três réplicas espalhadas por zonas, PDB e autoscaling
- [ ] Probes de startup, readiness e liveness corretas
- [ ] `requests` baseados em medição e limite de memória
- [ ] Encerramento gracioso ao receber `SIGTERM`
- [ ] Timeouts, retries e circuit breaker nas chamadas síncronas; consumidores idempotentes

**Segurança**

- [ ] Workload Identity e Key Vault; nenhum segredo em código, imagem ou ConfigMap
- [ ] Imagens assinadas, sem vulnerabilidades críticas, de registro privado
- [ ] Pod Security `restricted`, network policies de negação padrão
- [ ] Azure Policy, Defender for Containers e logs de auditoria ativos

**Operação**

- [ ] Métricas, logs e traces correlacionados; dashboards por serviço
- [ ] SLOs definidos, alertas com dono e runbook
- [ ] Backups testados, RTO e RPO definidos e validados
- [ ] GitOps como único caminho de mudança; rollback documentado
- [ ] Orçamentos, análise de custos por namespace e revisão periódica de dimensionamento

### 17.3 Trilha de estudos

1. **Kubernetes a fundo:** agendamento, rede (CNI, DNS, Services), armazenamento, controladores e operadores.
2. **Arquitetura de microsserviços:** *Building Microservices* (Sam Newman), *Domain-Driven Design* (Eric Evans) e o **Azure Architecture Center**, com os guias de microsserviços e a arquitetura de referência de baseline do AKS.
3. **Well-Architected Framework:** os cinco pilares (confiabilidade, segurança, custo, excelência operacional e desempenho) e o guia específico para AKS.
4. **Plataforma interna:** Backstage, templates de serviço e *golden paths* para que times criem serviços novos já dentro dos padrões.
5. **Observabilidade e SRE:** OpenTelemetry, SLOs e o livro *Site Reliability Engineering* do Google.
6. **Alternativas:** Azure Container Apps, para entender quando um serviço não precisa de AKS.

### 17.4 Certificações relacionadas

| Certificação | Emissor | Foco |
| --- | --- | --- |
| KCNA | CNCF / Linux Foundation | Fundamentos de Kubernetes e cloud native |
| CKAD | CNCF / Linux Foundation | Desenvolver e implantar aplicações no Kubernetes (prova prática) |
| CKA | CNCF / Linux Foundation | Administrar clusters Kubernetes (prova prática) |
| CKS | CNCF / Linux Foundation | Segurança de Kubernetes; exige a CKA |
| AZ-104 | Microsoft | Administração do Azure, incluindo AKS e rede |
| AZ-305 | Microsoft | Arquitetura de soluções no Azure |
| AZ-400 | Microsoft | DevOps no Azure: CI/CD, GitOps, segurança e monitoramento |

O catálogo de certificações da Microsoft muda com frequência; confirme no **Microsoft Learn** quais exames estão ativos e o conteúdo atual de cada um antes de se preparar.

## 18. Apêndices

### A. Comandos `az aks` mais usados

| Tarefa | Comando |
| --- | --- |
| Versões disponíveis | `az aks get-versions -l brazilsouth -o table` |
| Credenciais do cluster | `az aks get-credentials -g $RG -n $CLUSTER` |
| Estado e configuração | `az aks show -g $RG -n $CLUSTER -o yaml` |
| Upgrades disponíveis | `az aks get-upgrades -g $RG -n $CLUSTER -o table` |
| Atualizar o Kubernetes | `az aks upgrade -g $RG -n $CLUSTER --kubernetes-version <versão>` |
| Atualizar só a imagem dos nós | `az aks upgrade -g $RG -n $CLUSTER --node-image-only` |
| Listar node pools | `az aks nodepool list -g $RG --cluster-name $CLUSTER -o table` |
| Escalar um pool | `az aks nodepool scale -g $RG --cluster-name $CLUSTER -n apps --node-count 5` |
| Parar e iniciar (dev) | `az aks stop -g $RG -n $CLUSTER` / `az aks start -g $RG -n $CLUSTER` |
| Vincular ao ACR | `az aks update -g $RG -n $CLUSTER --attach-acr <acr>` |
| Verificar acesso ao ACR | `az aks check-acr -g $RG -n $CLUSTER --acr <acr>.azurecr.io` |
| Habilitar add-ons | `az aks enable-addons -g $RG -n $CLUSTER --addons azure-policy,azure-keyvault-secrets-provider` |
| Habilitar KEDA | `az aks update -g $RG -n $CLUSTER --enable-keda` |
| Habilitar Gateway API e app routing | `az aks update -g $RG -n $CLUSTER --enable-gateway-api --enable-app-routing-istio` |
| Comando em cluster privado | `az aks command invoke -g $RG -n $CLUSTER --command "kubectl get pods -A"` |

### B. `kubectl` para o dia a dia

| Tarefa | Comando |
| --- | --- |
| Pods com nó e IP | `kubectl get pods -n pedidos -o wide` |
| Detalhes e eventos | `kubectl describe pod <pod> -n pedidos` |
| Logs, inclusive da execução anterior | `kubectl logs <pod> -n pedidos --previous` |
| Logs de todas as réplicas | `kubectl logs -l app.kubernetes.io/name=pedidos-api -n pedidos --tail=100` |
| Eventos recentes | `kubectl get events -A --sort-by=.lastTimestamp` |
| Consumo | `kubectl top pods -A --sort-by=cpu` |
| Status do rollout | `kubectl rollout status deploy/pedidos-api -n pedidos` |
| Desfazer o rollout (só emergência; com GitOps, use `git revert`) | `kubectl rollout undo deploy/pedidos-api -n pedidos` |
| Rotas da Gateway API | `kubectl get gateway,httproute -A` |
| Depuração com contêiner efêmero | `kubectl debug -it <pod> --image=busybox:1.36 --target=api` |
| Esvaziar um nó | `kubectl drain <nó> --ignore-daemonsets --delete-emptydir-data` |
| Testar permissões | `kubectl auth can-i create deployments -n pedidos` |

### C. Glossário

| Termo | Definição |
| --- | --- |
| ACR | Azure Container Registry, o registro privado de imagens e charts |
| ADR | Architecture Decision Record: registro curto de uma decisão de arquitetura |
| AGC | Application Gateway for Containers, balanceador de camada 7 gerenciado para o AKS |
| AKS Automatic | Modo do AKS com configuração de produção pronta e nós provisionados automaticamente |
| APIM | Azure API Management, gateway e governança de APIs |
| Bounded context | Fronteira de um modelo de domínio no DDD; bom candidato a microsserviço |
| Canary | Implantação que envia uma pequena parte do tráfego para a nova versão |
| CNI Overlay | Modelo de rede do AKS em que os Pods usam um espaço de IP separado da VNet |
| Consistência eventual | Os dados de serviços diferentes ficam coerentes depois de um intervalo, não instantaneamente |
| Dapr | Runtime com blocos prontos (pub/sub, estado, workflow) para microsserviços |
| Dead-letter queue | Fila que recebe mensagens que não puderam ser processadas |
| Gateway API | API do Kubernetes que sucede o Ingress para roteamento de tráfego |
| GitOps | O Git como fonte da verdade, aplicado continuamente ao cluster por um agente |
| HPA | Horizontal Pod Autoscaler: escala o número de Pods por métricas |
| Idempotência | Processar a mesma mensagem duas vezes produz o mesmo resultado que uma vez |
| KEDA | Autoscaler orientado a eventos, capaz de escalar a zero |
| Landing zone | Ambiente do Azure pré-configurado com rede, identidade e políticas |
| NAP | Node auto-provisioning: cria nós do tamanho certo automaticamente (baseado em Karpenter) |
| Outbox | Padrão que grava o evento na mesma transação do dado para publicá-lo com segurança |
| PDB | PodDisruptionBudget: mínimo de réplicas disponíveis durante manutenções |
| Private Endpoint | Endereço privado na VNet para um serviço PaaS do Azure |
| Saga | Sequência de transações locais com compensações, no lugar de uma transação distribuída |
| Service mesh | Camada de proxies que controla e protege o tráfego entre serviços |
| SLO | Objetivo de nível de serviço, como 99,9% de requisições bem-sucedidas |
| Workload Identity | Federação que permite a um Pod acessar o Azure sem segredos |

### D. Fontes

- [Supported Kubernetes versions in AKS](https://learn.microsoft.com/azure/aks/supported-kubernetes-versions) — calendário de versões e política de suporte
- [Migrate from application routing NGINX to Gateway API](https://learn.microsoft.com/azure/aks/app-routing-nginx-to-gateway-api-migration) — aposentadoria do Ingress-NGINX e migração
- [Application routing add-on migration](https://learn.microsoft.com/azure/aks/app-routing-migration) — fim do suporte ao NGINX gerenciado em novembro de 2026
- [Application routing Gateway API](https://learn.microsoft.com/azure/aks/app-routing-gateway-api) — Gateway API como padrão do AKS Automatic 1.36+
- [Blog do AKS: Gateway API no application routing](https://blog.aks.azure.com/authors/jaiveer-katariya) — disponibilidade geral em abril de 2026
- [AKS Automatic: anúncio de disponibilidade geral](https://azure.microsoft.com/blog/azure-kubernetes-service-automatic-fast-and-frictionless-kubernetes-for-all/)
- [Node auto-provisioning: disponibilidade geral](https://mc.merill.net/message/AZ498247) e [documentação](https://learn.microsoft.com/azure/aks/node-autoprovision)

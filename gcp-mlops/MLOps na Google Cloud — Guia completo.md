# MLOps na Google Cloud — Guia completo

Nov 20, 2026 · @Oi

## 1. Introdução ao MLOps

**MLOps** é o conjunto de práticas que leva modelos de machine learning do notebook para a produção de forma confiável, repetível e auditável, e os mantém saudáveis depois disso. Na Google Cloud, a base é o **Gemini Enterprise Agent Platform** (o antigo **Vertex AI**), apoiado por BigQuery, Cloud Storage, Artifact Registry, Cloud Build e o restante da plataforma.

> **Mudança de nome (abril de 2026):** no Google Cloud Next '26, em 22 de abril de 2026, o Vertex AI passou a se chamar **Gemini Enterprise Agent Platform**, ou simplesmente **Agent Platform**. Todos os serviços do Vertex AI continuam existindo e o roadmap passa a ser entregue pela Agent Platform; links antigos redirecionam automaticamente ([anúncio oficial](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform), [notas de versão](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes)). A mudança é de marca: nos exemplos de código deste guia, os identificadores técnicos continuam `google-cloud-aiplatform`, `aiplatform.googleapis.com` e `gcloud ai`, que são os que a documentação ainda usa. Como muito material de estudo e muitas vagas ainda dizem “Vertex AI”, o guia mostra os dois nomes na primeira menção.

| Nome antigo (Vertex AI) | Nome atual (Agent Platform) |
| --- | --- |
| Vertex AI Workbench | Agent Platform Workbench |
| Colab Enterprise | Agent Platform Colab Enterprise |
| Vertex AI Experiments | Experiments on Agent Platform |
| Vertex AI Training | Agent Platform Managed Training |
| Vertex AI Feature Store | Agent Platform Feature Store |
| Vertex AI Model Registry | Agent Platform Model Registry |
| Vertex AI Model Evaluation | Agent Platform Model Evaluation |
| Vertex AI Pipelines | Agent Platform Pipelines |
| Vertex AI Endpoints e Prediction | Agent Platform Endpoints e Agent Platform Inference |
| Vertex AI Batch Prediction | Agent Platform Batch Inference |
| Vertex AI Model Monitoring | Model Monitoring on Agent Platform |
| Vertex AI Studio | Agent Studio |
| Gen AI evaluation service | Gen AI evals |
| Vertex AI Agent Engine | Agent Runtime |
| Vertex AI RAG Engine e Vector Search | RAG Engine e Vector Search |

### 1.1 Por que MLOps não é só DevOps

Um sistema de ML tem três coisas que mudam, e não apenas uma: **código**, **dados** e **modelo**. Um modelo pode piorar sem que ninguém altere uma linha de código, simplesmente porque o mundo mudou e os dados de produção deixaram de se parecer com os de treinamento.

| Aspecto | DevOps tradicional | MLOps |
| --- | --- | --- |
| O que é versionado | Código e configuração | Código, dados, features, hiperparâmetros e modelos |
| O que é testado | Comportamento do código | Código, qualidade dos dados, desempenho e viés do modelo |
| O que é entregue | Uma aplicação | Um pipeline que treina e entrega modelos |
| Por que algo quebra | Bug ou falha de infraestrutura | Também por *drift* de dados ou de conceito |
| Ciclo | Build → teste → deploy | Dados → treino → avaliação → deploy → monitoramento → retreino |

Na prática, o código do modelo costuma ser uma parte pequena do sistema. Coleta e validação de dados, engenharia de features, infraestrutura de serving, monitoramento e governança são o grosso do trabalho, e é disso que o MLOps trata.

### 1.2 Níveis de maturidade

O Google descreve três níveis de maturidade de MLOps. Saber onde você está ajuda a decidir o próximo investimento:

| Nível | Como funciona | Sinal típico | Próximo passo |
| --- | --- | --- | --- |
| **0 — Manual** | Cientista treina em notebook e entrega um arquivo de modelo para a engenharia publicar | Ninguém consegue reproduzir o modelo que está em produção | Colocar o treino em código versionado e registrar experimentos |
| **1 — Pipeline automatizado** | O treino roda em um pipeline orquestrado, com validação de dados e de modelo; o retreino é disparado por agenda ou por dados novos | O modelo é retreinado sozinho, mas mudar o pipeline ainda é manual | Testar e implantar o próprio pipeline via CI/CD |
| **2 — CI/CD de pipelines** | Mudanças de código passam por testes, constroem componentes e implantam novos pipelines automaticamente em cada ambiente | Uma mudança de feature vai do commit à produção sem passos manuais, com aprovação onde necessário | Otimizar custo, governança e monitoramento contínuo |

Este guia leva você do nível 0 ao nível 2, capítulo a capítulo.

### 1.3 Princípios que guiam o guia

- **Tudo como código:** infraestrutura (Terraform), pipelines (KFP), configuração e políticas ficam no Git.
- **Reprodutibilidade:** qualquer modelo em produção deve ser rastreável até o código, os dados e os parâmetros que o geraram.
- **Automação com portões de qualidade:** um modelo só avança se passar em critérios objetivos de desempenho e de dados.
- **Ambientes isolados:** desenvolvimento, homologação e produção em projetos separados.
- **Observabilidade:** monitorar o modelo como se monitora um serviço, mais a qualidade das previsões.
- **Menor privilégio:** cada componente roda com uma service account dedicada e permissões mínimas.

### 1.4 Pré-requisitos e como usar este guia

Você vai precisar de Python intermediário, noções de machine learning (treino, validação, métricas), familiaridade com Docker e Git, e um projeto na Google Cloud com faturamento ativo. Os exemplos usam a **Google Cloud CLI** (`gcloud`), o **SDK do Vertex AI para Python** (`google-cloud-aiplatform`) e o **Kubeflow Pipelines SDK** (`kfp`).

> **Atenção:** treinamentos com GPU, endpoints online e alguns serviços geram custo enquanto estiverem ativos. Ao final de cada laboratório, apague endpoints e recursos que não for usar, e configure um alerta de orçamento antes de começar (capítulo 3).

Cada capítulo termina com um **laboratório**. Eles se encadeiam em um caso único: um modelo de **previsão de churn** de clientes, que começa como notebook e termina como sistema monitorado com retreino automático.

1. Introdução ao MLOps
2. Panorama dos serviços e arquitetura de referência
3. Fundação: projetos, IAM, rede, Terraform e custos
4. Dados e features
5. Experimentação
6. Treinamento
7. Model Registry e avaliação
8. Vertex AI Pipelines
9. Implantação e serving
10. CI/CD para ML
11. Monitoramento e retreino
12. GenAI e LLMOps
13. Segurança, governança e custos
14. Projeto final, checklist de produção e trilha de estudos
15. Apêndices

## 2. Panorama dos serviços e arquitetura de referência

Um sistema de MLOps na Google Cloud tem cinco blocos em sequência — dados, experimentação, pipeline, registro e serving — mais o monitoramento, que realimenta o pipeline. Por baixo de tudo ficam o CI/CD e a fundação de segurança.

&#91;embedded content: Arquitetura de referência de MLOps na Google Cloud\]

O **pipeline** é o centro: ele transforma dados validados em um modelo avaliado e registrado. O CI/CD não implanta um modelo diretamente; implanta o **pipeline** que produz modelos. Quando o monitoramento detecta *drift*, dispara o mesmo pipeline para retreinar.

### 2.1 Serviços por etapa do ciclo de vida

| Etapa | Serviços principais | Capítulo |
| --- | --- | --- |
| Fundação | Resource Manager (pastas e projetos), IAM, VPC, Secret Manager, Cloud KMS, Terraform | 3 |
| Dados | BigQuery, Cloud Storage, Dataflow, Dataproc, Dataplex, Agent Platform Feature Store | 4 |
| Experimentação | Agent Platform Workbench, Colab Enterprise, Experiments, TensorBoard | 5 |
| Treinamento | Agent Platform Managed Training (custom jobs, tuning de hiperparâmetros), AutoML, BigQuery ML | 6 |
| Registro e avaliação | Agent Platform Model Registry, Model Evaluation, ML Metadata | 7 |
| Orquestração | Agent Platform Pipelines (Kubeflow Pipelines), Cloud Scheduler, Eventarc | 8 |
| Serving | Agent Platform Endpoints, Online e Batch Inference, Cloud Run, GKE | 9 |
| CI/CD | Cloud Build, Artifact Registry, Cloud Deploy, GitHub Actions com Workload Identity Federation | 10 |
| Monitoramento | Model Monitoring on Agent Platform, Cloud Logging, Cloud Monitoring | 11 |
| GenAI | Gemini e Model Garden, Agent Studio, Gen AI evals, RAG Engine, Vector Search, Agent Runtime | 12 |

### 2.2 Qual forma de treinar escolher

| Opção | Quando usar | Esforço | Controle |
| --- | --- | --- | --- |
| **BigQuery ML** | Dados já estão no BigQuery e o problema é clássico (regressão, classificação, séries temporais, clustering); equipe forte em SQL | Baixo: só SQL | Moderado |
| **AutoML** | Dados tabulares, imagem ou texto e pouco tempo para engenharia de modelos | Baixo | Baixo |
| **Custom training** | Você precisa de um framework específico (scikit-learn, XGBoost, PyTorch, TensorFlow), GPUs ou lógica própria | Médio a alto | Total |
| **Modelos fundacionais (Gemini, Model Garden)** | Tarefas de linguagem, visão ou multimodais em que um modelo pré-treinado com prompt, RAG ou tuning resolve | Baixo a médio | Variável |

Comece pela opção mais simples que atende ao requisito. Um baseline em BigQuery ML entregue em uma semana costuma valer mais que um modelo sofisticado entregue em três meses, e serve de régua para qualquer evolução.

### 2.3 Regiões

Mantenha dados, treinamento e serving na **mesma região** para reduzir latência e custos de transferência e para atender a requisitos de residência de dados, como os da LGPD. Para cargas no Brasil, avalie `southamerica-east1` (São Paulo); confira na documentação quais recursos e tipos de GPU estão disponíveis nela antes de escolher, porque a oferta varia por região. Os exemplos deste guia usam variáveis (`$REGION`) para facilitar a troca.

### Laboratório

- [ ] Desenhe, para o caso de churn, quais serviços você usaria em cada bloco do diagrama e justifique cada escolha.
- [ ] Liste os requisitos não funcionais do caso: latência da previsão (online ou em lote?), frequência de retreino, região e restrições de dados pessoais.
- [ ] Escolha entre BigQuery ML, AutoML e custom training para o baseline e registre a decisão.

## 3. Fundação: projetos, IAM, rede, Terraform e custos

Antes do primeiro modelo, defina onde cada coisa roda e quem pode fazer o quê. Uma fundação bem feita evita os dois problemas mais caros de corrigir depois: permissões amplas demais e ambientes misturados.

### 3.1 Organização de projetos e ambientes

Use **um projeto por ambiente**, agrupados em uma pasta. O isolamento por projeto separa IAM, cotas, faturamento e logs:

| Projeto | Para quê | Quem tem acesso amplo |
| --- | --- | --- |
| `ml-dev` | Experimentação, notebooks, pipelines em teste | Cientistas e engenheiros de ML |
| `ml-stg` | Pipelines e endpoints idênticos aos de produção, com dados de homologação | Somente o CI/CD; pessoas em leitura |
| `ml-prd` | Pipelines de treino, endpoints e monitoramento de produção | Somente o CI/CD; acesso humano emergencial e auditado |
| `ml-shared` (opcional) | Artifact Registry, repositório de modelos aprovados, Terraform state | Plataforma |

Dados de produção normalmente vivem em projetos de dados próprios; os projetos de ML recebem acesso de leitura a datasets específicos do BigQuery.

### 3.2 APIs e configuração inicial

```bash
export PROJECT_ID=ml-dev-123456
export REGION=southamerica-east1
gcloud config set project $PROJECT_ID

gcloud services enable \
  aiplatform.googleapis.com \
  bigquery.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  secretmanager.googleapis.com

# bucket para artefatos de pipelines e modelos (regional, com acesso uniforme)
gcloud storage buckets create gs://$PROJECT_ID-ml-artifacts \
  --location=$REGION --uniform-bucket-level-access

# repositório de imagens Docker
gcloud artifacts repositories create ml-images \
  --repository-format=docker --location=$REGION
```

No Python, inicialize o SDK uma vez no início de cada script:

```python
from google.cloud import aiplatform

aiplatform.init(
    project="ml-dev-123456",
    location="southamerica-east1",
    staging_bucket="gs://ml-dev-123456-ml-artifacts",
    experiment="churn",          # opcional: associa execuções a um experimento
)
```

### 3.3 IAM e service accounts

Cada carga de trabalho roda com sua **própria service account**, com o mínimo de papéis. Nunca use a service account padrão do Compute Engine, que costuma ter permissões amplas demais.

| Service account | Usada por | Papéis típicos |
| --- | --- | --- |
| `sa-pipelines` | Execuções de pipelines e jobs de treino | `roles/aiplatform.user`, `roles/bigquery.dataViewer` (datasets específicos), `roles/bigquery.jobUser`, `roles/storage.objectAdmin` (bucket de artefatos) |
| `sa-endpoint` | Contêineres de serving | `roles/storage.objectViewer` (artefatos do modelo), `roles/logging.logWriter` |
| `sa-cicd` | Cloud Build ou GitHub Actions | `roles/aiplatform.user`, `roles/artifactregistry.writer`, `roles/iam.serviceAccountUser` sobre `sa-pipelines` |
| `sa-monitoring` | Jobs de monitoramento e alertas | `roles/aiplatform.user`, `roles/bigquery.dataViewer` |

```bash
gcloud iam service-accounts create sa-pipelines --display-name="Pipelines de ML"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:sa-pipelines@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

Para pessoas, conceda papéis a **grupos**, não a usuários individuais. Em vez de chaves JSON de service account, use **impersonation** (`gcloud ... --impersonate-service-account`) localmente e **Workload Identity Federation** no CI/CD (capítulo 10). Aplique a política de organização `iam.disableServiceAccountKeyCreation` para impedir a criação de chaves.

### 3.4 Rede e segurança

- **VPC Service Controls:** cria um perímetro em volta dos projetos de dados e ML, impedindo a exfiltração de dados para fora dele, mesmo com credenciais válidas (capítulo 13).
- **Private Service Connect e peering:** permitem que jobs de treino, pipelines e endpoints acessem recursos na sua VPC (bancos, APIs internas) sem IP público.
- **Endpoints privados:** expõem modelos apenas dentro da VPC.
- **Secret Manager:** guarda credenciais de terceiros; nunca coloque segredos em código, imagens ou parâmetros de pipeline.
- **CMEK:** chaves de criptografia gerenciadas por você no Cloud KMS, quando a política de compliance exigir.

### 3.5 Infraestrutura como código com Terraform

Tudo o que foi criado acima deve existir como código, para que `ml-stg` e `ml-prd` sejam cópias fiéis de `ml-dev`:

```hcl
variable "project_id" {}
variable "region" { default = "southamerica-east1" }

resource "google_project_service" "apis" {
  for_each = toset([
    "aiplatform.googleapis.com",
    "artifactregistry.googleapis.com",
    "cloudbuild.googleapis.com",
  ])
  project            = var.project_id
  service            = each.value
  disable_on_destroy = false
}

resource "google_service_account" "pipelines" {
  project      = var.project_id
  account_id   = "sa-pipelines"
  display_name = "Pipelines de ML"
}

resource "google_project_iam_member" "pipelines_aiplatform" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_service_account.pipelines.email}"
}

resource "google_storage_bucket" "artifacts" {
  project                     = var.project_id
  name                        = "${var.project_id}-ml-artifacts"
  location                    = var.region
  uniform_bucket_level_access = true
}

resource "google_artifact_registry_repository" "images" {
  project       = var.project_id
  location      = var.region
  repository_id = "ml-images"
  format        = "DOCKER"
}
```

Guarde o estado do Terraform em um bucket com versionamento, use um diretório ou *workspace* por ambiente e aplique as mudanças pelo CI/CD.

### 3.6 Controle de custos desde o primeiro dia

- Crie um **orçamento com alertas** (por exemplo, 50%, 90% e 100%) em cada projeto.
- Exporte o faturamento para o **BigQuery** e use **labels** em todos os recursos (`env`, `team`, `model`) para saber quanto custa cada modelo.
- Endpoints online cobram por nó enquanto estiverem implantados, mesmo sem tráfego: desfaça a implantação de endpoints de teste.
- Notebooks do Workbench devem ter **desligamento por inatividade** configurado.
- Prefira VMs **spot** para treinos tolerantes a interrupção.

### Laboratório

- [ ] Crie os projetos `ml-dev` e `ml-prd` (ou simule com um só projeto e prefixos), habilite as APIs e crie bucket e repositório.
- [ ] Escreva o Terraform da seção 3.5 e aplique-o no ambiente de desenvolvimento.
- [ ] Crie a `sa-pipelines` com os papéis mínimos e teste a impersonation com `gcloud storage ls --impersonate-service-account`.
- [ ] Configure um orçamento com alerta por e-mail e a exportação de faturamento para o BigQuery.

## 4. Dados e features

A maior parte dos problemas de modelos em produção nasce nos dados. Na Google Cloud, o **BigQuery** é o centro: guarda os dados brutos e tratados, calcula features em SQL e serve de fonte para treinamento, Feature Store e monitoramento.

### 4.1 Onde cada tipo de dado fica

| Serviço | Uso em ML |
| --- | --- |
| BigQuery | Dados tabulares, features, conjuntos de treino e avaliação, logs de previsões |
| Cloud Storage | Arquivos (imagens, áudio, Parquet, TFRecord), artefatos de modelos e de pipelines |
| Dataflow (Apache Beam) | Processamento em lote e streaming em grande escala, inclusive features em tempo real |
| Dataproc / Serverless for Apache Spark | Cargas Spark existentes |
| Pub/Sub | Eventos em tempo real que alimentam features ou disparam pipelines |
| Dataplex | Catálogo, linhagem e qualidade de dados |

### 4.2 Preparando o conjunto de treino no BigQuery

Para o caso de churn, as features são calculadas a partir de uma **data de corte**: tudo o que se sabe sobre o cliente até aquela data, e o rótulo é o que aconteceu depois dela.

```sql
CREATE OR REPLACE TABLE `ml_churn.treino_2026_10` AS
WITH base AS (
  SELECT
    c.cliente_id,
    DATE '2026-10-01' AS data_corte,
    DATE_DIFF(DATE '2026-10-01', c.data_cadastro, DAY) AS dias_cliente,
    COUNTIF(p.data_pedido BETWEEN DATE '2026-07-01' AND DATE '2026-09-30') AS pedidos_90d,
    SUM(IF(p.data_pedido BETWEEN DATE '2026-07-01' AND DATE '2026-09-30', p.valor, 0)) AS gasto_90d,
    DATE_DIFF(DATE '2026-10-01', MAX(p.data_pedido), DAY) AS dias_desde_ultimo_pedido
  FROM `dados.clientes` c
  LEFT JOIN `dados.pedidos` p
    ON p.cliente_id = c.cliente_id AND p.data_pedido < DATE '2026-10-01'
  GROUP BY c.cliente_id, c.data_cadastro
)
SELECT
  b.*,
  -- rótulo: não comprou nos 60 dias seguintes à data de corte
  NOT EXISTS (
    SELECT 1 FROM `dados.pedidos` p
    WHERE p.cliente_id = b.cliente_id
      AND p.data_pedido BETWEEN DATE '2026-10-01' AND DATE '2026-11-29'
  ) AS churn
FROM base b;
```

> **Atenção: vazamento de dados (*data leakage*).** Nenhuma feature pode usar informação posterior à data de corte. Um modelo com vazamento parece excelente na avaliação e falha em produção. Filtre sempre por `< data_corte` ao calcular features.

Divida treino, validação e teste **por tempo** (meses anteriores para treino, posteriores para teste) quando o modelo for usado para prever o futuro; divisão aleatória mistura passado e futuro.

### 4.3 Qualidade de dados

Trate a validação de dados como um teste automatizado que roda **antes** de todo treino:

- **Esquema:** colunas esperadas, tipos e nulidade.
- **Faixas e domínios:** `idade` entre 18 e 120, `estado` dentro da lista de UFs.
- **Volume:** número de linhas dentro de uma faixa esperada (uma queda de 40% indica falha de ingestão).
- **Distribuição:** comparação com a execução anterior para pegar mudanças bruscas.
- **Rótulo:** proporção de classes plausível.

Ferramentas: verificações em SQL dentro do pipeline (simples e eficaz), **Dataplex Data Quality**, **TensorFlow Data Validation** ou **Great Expectations**. Se a validação falhar, o pipeline para e alerta, em vez de treinar com dados ruins.

```sql
-- exemplo de verificação que o pipeline executa e compara com limites
SELECT
  COUNT(*) AS linhas,
  COUNTIF(dias_cliente IS NULL) / COUNT(*) AS pct_nulos_dias_cliente,
  AVG(CAST(churn AS INT64)) AS taxa_churn
FROM `ml_churn.treino_2026_10`;
```

### 4.4 Versionamento de dados

Para reproduzir um modelo, você precisa saber **exatamente** com quais dados ele foi treinado. Estratégias no BigQuery:

1. **Tabelas imutáveis por execução**, com a data no nome (`treino_2026_10`), nunca sobrescritas.
2. **Snapshots de tabela** (`CREATE SNAPSHOT TABLE ... CLONE ...`), baratos porque só armazenam as diferenças.
3. **Time travel** (`FOR SYSTEM_TIME AS OF`), que permite consultar o estado de uma tabela em um momento recente (até 7 dias, por padrão).
4. Registrar no experimento e nos metadados do pipeline a tabela, a consulta e o momento usados.

Para arquivos no Cloud Storage, use caminhos versionados (`gs://.../datasets/churn/v=2026-10-01/`) e, se quiser, ative o versionamento de objetos do bucket.

### 4.5 Agent Platform Feature Store

Uma **Feature Store** resolve dois problemas: reaproveitar features entre modelos e servir, em produção, **exatamente** os mesmos valores usados no treino, eliminando o *training-serving skew*.

A versão atual da Feature Store (antiga Vertex AI Feature Store) usa o **BigQuery como armazenamento offline**: você registra tabelas ou views do BigQuery como **feature groups** e cria **feature views** que sincronizam os valores mais recentes para uma **loja online** de baixa latência. A versão legada (v1) e o modo de serving online “Optimized” foram descontinuados, então projetos novos devem usar a versão atual com serving online baseado em Bigtable ([notas de versão](https://docs.cloud.google.com/vertex-ai/docs/core-release-notes)).

| Conceito | O que é |
| --- | --- |
| Feature group | Registro de uma fonte do BigQuery (tabela ou view) com a coluna de entidade e de timestamp |
| Feature | Uma coluna dessa fonte |
| Online store | Infraestrutura de serving de baixa latência |
| Feature view | Conjunto de features sincronizado da fonte para a online store, sob agenda ou demanda |

```python
# leitura online de features para um cliente, no momento da previsão
from google.cloud.aiplatform_v1 import FeatureOnlineStoreServiceClient
from google.cloud.aiplatform_v1.types import feature_online_store_service as fos

client = FeatureOnlineStoreServiceClient(
    client_options={"api_endpoint": f"{REGION}-aiplatform.googleapis.com"})
resposta = client.fetch_feature_values(
    request=fos.FetchFeatureValuesRequest(
        feature_view=(f"projects/{PROJECT_ID}/locations/{REGION}/featureOnlineStores/"
                      "loja_online/featureViews/churn_cliente"),
        data_key=fos.FeatureViewDataKey(key="cliente-123"),
    ))
```

Para treino, a vantagem da Feature Store baseada em BigQuery é que os dados offline **já estão** no BigQuery: monte o conjunto de treino com SQL e um *point-in-time join* (as features de cada cliente como estavam na data de cada exemplo).

> **Quando não usar:** se o modelo só faz previsões em lote e calcula as features no mesmo pipeline, uma tabela do BigQuery bem versionada basta. A Feature Store vale a pena quando há serving online com features pré-calculadas ou muitas features compartilhadas entre equipes.

### Laboratório

- [ ] Carregue no BigQuery um dataset público de churn (por exemplo, de telecom) ou gere dados sintéticos de clientes e pedidos.
- [ ] Escreva a consulta de features com data de corte e crie conjuntos de treino e teste separados por tempo.
- [ ] Escreva três verificações de qualidade em SQL com limites e um script Python que falhe se algum limite for violado.
- [ ] Crie um snapshot da tabela de treino e anote o nome dele, para usá-lo no capítulo 5.

## 5. Experimentação

A experimentação é onde o cientista explora dados e testa abordagens. O objetivo de MLOps aqui não é burocratizar, e sim garantir que **toda execução seja registrada**: parâmetros, dados, métricas e artefatos, para que o melhor resultado possa ser reproduzido e levado para o pipeline.

### 5.1 Ambientes de notebook

| Opção | Perfil | Pontos fortes |
| --- | --- | --- |
| **Agent Platform Workbench** (antigo Vertex AI Workbench) | Instâncias JupyterLab gerenciadas, com controle da VM, GPUs e ambientes | Customização, integração com BigQuery e Git, execuções agendadas de notebooks |
| **Colab Enterprise** | Notebooks sem servidor para gerenciar, com runtimes sob demanda | Início rápido, colaboração, também disponível dentro do console do BigQuery |
| IDE local + SDK | Desenvolvimento em VS Code ou PyCharm, enviando jobs para a nuvem | Melhor para código de produção, testes e revisão |

Configure desligamento automático por inatividade, use a service account de desenvolvimento e mantenha os notebooks em um repositório Git.

> **Dica:** notebook é ótimo para explorar e ruim para produção. Assim que uma etapa estabilizar (preparação de dados, treino, avaliação), mova-a para um módulo Python com testes. O notebook passa a apenas chamar essas funções, e o pipeline do capítulo 8 reutiliza o mesmo código.

### 5.2 Experiments on Agent Platform

O **Experiments** (antigo Vertex AI Experiments) registra cada execução (*run*) com parâmetros, métricas, séries temporais e artefatos, e permite comparar execuções lado a lado no console. Os dados ficam no ML Metadata, o mesmo repositório de linhagem usado pelos pipelines.

```python
from google.cloud import aiplatform
import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score, average_precision_score

aiplatform.init(project=PROJECT_ID, location=REGION, experiment="churn")

treino = pd.read_gbq("SELECT * FROM ml_churn.treino_2026_10", project_id=PROJECT_ID)
teste = pd.read_gbq("SELECT * FROM ml_churn.teste_2026_10", project_id=PROJECT_ID)
features = ["dias_cliente", "pedidos_90d", "gasto_90d", "dias_desde_ultimo_pedido"]

for profundidade in [2, 3, 4]:
    with aiplatform.start_run(run=f"gb-prof-{profundidade}"):
        params = {"n_estimators": 300, "max_depth": profundidade, "learning_rate": 0.05}
        aiplatform.log_params({**params, "dados_treino": "ml_churn.treino_2026_10"})

        modelo = GradientBoostingClassifier(**params).fit(treino[features], treino["churn"])
        prob = modelo.predict_proba(teste[features])[:, 1]

        aiplatform.log_metrics({
            "auc_roc": roc_auc_score(teste["churn"], prob),
            "auc_pr": average_precision_score(teste["churn"], prob),
        })
```

Recupere e compare todas as execuções como um DataFrame:

```python
df = aiplatform.get_experiment_df("churn")
df.sort_values("metric.auc_pr", ascending=False).head()
```

### 5.3 TensorBoard

Para redes neurais, associe um **TensorBoard gerenciado** ao experimento e acompanhe curvas de perda e métricas por época durante treinos longos, inclusive dos jobs remotos do capítulo 6:

```python
tb = aiplatform.Tensorboard.create(display_name="churn-tb")
aiplatform.init(experiment="churn", experiment_tensorboard=tb)
```

### 5.4 O que registrar em cada execução

| Categoria | Exemplos |
| --- | --- |
| Código | Hash do commit Git, nome da imagem de contêiner |
| Dados | Tabela ou snapshot do BigQuery, intervalo de datas, número de linhas, taxa do rótulo |
| Parâmetros | Hiperparâmetros, lista de features, semente aleatória |
| Métricas | Métrica principal e secundárias, métricas por segmento (região, canal, faixa de cliente) |
| Artefatos | Modelo serializado, gráficos de importância de features e matriz de confusão |

### 5.5 Escolhendo a métrica certa

Defina a métrica principal com o negócio **antes** de experimentar. Para churn, a classe positiva costuma ser rara (por exemplo, 5% dos clientes), então a acurácia engana: um modelo que diz “ninguém vai sair” acerta 95%. Prefira **AUC-PR**, **precisão e recall no ponto de corte usado pela operação** ou uma métrica de valor (receita retida pelas ações de retenção menos o custo delas). Sempre compare com um **baseline** simples, como uma regra de negócio ou uma regressão logística.

### Laboratório

- [ ] Crie uma instância do Workbench (ou um notebook no Colab Enterprise) com desligamento por inatividade.
- [ ] Treine uma regressão logística como baseline e três variações de gradient boosting, registrando tudo no Experiments.
- [ ] Compare as execuções no console e com `get_experiment_df`, e escolha a candidata justificando pela AUC-PR.
- [ ] Mova a preparação de dados e o treino do notebook para um pacote Python (`churn/dados.py`, `churn/treino.py`) com pelo menos um teste unitário cada.

## 6. Treinamento

Em produção, o treino não roda no notebook: roda como um **job gerenciado** do **Agent Platform Managed Training** (antigo Vertex AI Training), em máquinas criadas sob demanda e destruídas ao final. Você paga só pelo tempo de execução e o treino fica reproduzível, porque o ambiente é uma imagem de contêiner versionada.

### 6.1 Empacotando o treino em um contêiner

```text
churn/
├── Dockerfile
├── requirements.txt
└── trainer/
    ├── __init__.py
    └── task.py
```

```python
# trainer/task.py
import argparse, os, joblib
import pandas as pd
from google.cloud import bigquery, storage
from sklearn.ensemble import GradientBoostingClassifier

FEATURES = ["dias_cliente", "pedidos_90d", "gasto_90d", "dias_desde_ultimo_pedido"]

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--tabela-treino", required=True)
    p.add_argument("--max-depth", type=int, default=3)
    p.add_argument("--learning-rate", type=float, default=0.05)
    args = p.parse_args()

    df = bigquery.Client().query(f"SELECT * FROM `{args.tabela_treino}`").to_dataframe()
    modelo = GradientBoostingClassifier(
        n_estimators=300, max_depth=args.max_depth, learning_rate=args.learning_rate
    ).fit(df[FEATURES], df["churn"])

    # AIP_MODEL_DIR é definido pela plataforma: o diretório gs:// onde salvar o modelo
    destino = os.environ["AIP_MODEL_DIR"]
    joblib.dump(modelo, "model.joblib")
    bucket, prefixo = destino.removeprefix("gs://").split("/", 1)
    storage.Client().bucket(bucket).blob(f"{prefixo.rstrip('/')}/model.joblib") \
        .upload_from_filename("model.joblib")

if __name__ == "__main__":
    main()
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY trainer/ trainer/
ENTRYPOINT ["python", "-m", "trainer.task"]
```

```bash
export IMAGEM=$REGION-docker.pkg.dev/$PROJECT_ID/ml-images/churn-trainer:$(git rev-parse --short HEAD)
gcloud builds submit --tag $IMAGEM .
```

Use a **hash do commit como tag** da imagem, nunca apenas `latest`: assim cada modelo aponta para o código exato que o gerou.

### 6.2 Executando um custom job

```python
job = aiplatform.CustomContainerTrainingJob(
    display_name="churn-treino",
    container_uri=IMAGEM,
    model_serving_container_image_uri=IMAGEM_SERVING,   # capítulo 9
)

modelo = job.run(
    args=["--tabela-treino", f"{PROJECT_ID}.ml_churn.treino_2026_10", "--max-depth", "3"],
    replica_count=1,
    machine_type="n2-standard-8",
    service_account=f"sa-pipelines@{PROJECT_ID}.iam.gserviceaccount.com",
    model_display_name="churn",
    labels={"env": "dev", "model": "churn"},
)
```

Ao final, o job já registra o modelo no Model Registry (capítulo 7). Para frameworks comuns, você também pode usar os **contêineres pré-construídos** de treino e serving do Google (scikit-learn, XGBoost, TensorFlow, PyTorch) e enviar apenas o seu pacote Python; confira os URIs atuais na lista oficial de contêineres pré-construídos.

### 6.3 GPUs, TPUs e VMs spot

```python
job.run(
    ...,
    machine_type="g2-standard-8",
    accelerator_type="NVIDIA_L4",
    accelerator_count=1,
)
```

| Recurso | Quando usar |
| --- | --- |
| CPU | Modelos tabulares clássicos (árvores, regressões) |
| GPU (L4, A100, H100 e outras) | Deep learning, fine-tuning de modelos de linguagem e visão |
| TPU | Treinos muito grandes em TensorFlow, JAX ou PyTorch/XLA |
| VMs spot | Treinos com checkpoint que toleram interrupção: custo bem menor |

A disponibilidade de cada acelerador varia por região. Se o treino for interrompível, salve **checkpoints** periódicos no Cloud Storage e retome a partir do último.

### 6.4 Tuning de hiperparâmetros

O serviço de tuning executa vários trials em paralelo e usa otimização bayesiana para escolher os próximos valores a testar. O código de treino só precisa reportar a métrica:

```python
# no final de trainer/task.py
import hypertune
hypertune.HyperTune().report_hyperparameter_tuning_metric(
    hyperparameter_metric_tag="auc_pr", metric_value=auc_pr)
```

```python
from google.cloud.aiplatform import hyperparameter_tuning as hpt

custom_job = aiplatform.CustomJob(
    display_name="churn-trial",
    worker_pool_specs=[{
        "machine_spec": {"machine_type": "n2-standard-8"},
        "replica_count": 1,
        "container_spec": {"image_uri": IMAGEM,
                           "args": ["--tabela-treino", TABELA]},
    }],
)

hp_job = aiplatform.HyperparameterTuningJob(
    display_name="churn-hpt",
    custom_job=custom_job,
    metric_spec={"auc_pr": "maximize"},
    parameter_spec={
        "max-depth": hpt.IntegerParameterSpec(min=2, max=8, scale="linear"),
        "learning-rate": hpt.DoubleParameterSpec(min=0.01, max=0.3, scale="log"),
    },
    max_trial_count=20,
    parallel_trial_count=4,
)
hp_job.run()
```

### 6.5 Treino distribuído

Para modelos grandes, defina vários *worker pools* (um chefe, workers e, se necessário, servidores de parâmetros). A plataforma injeta a variável `CLUSTER_SPEC` com a topologia, e frameworks como PyTorch DDP e `tf.distribute` usam essa informação. Para cargas muito grandes e recorrentes, existem os **Managed Training Clusters** (capacidade reservada) e o **Ray on Agent Platform**.

### 6.6 BigQuery ML: treinar sem sair do SQL

```sql
CREATE OR REPLACE MODEL `ml_churn.modelo_bqml`
OPTIONS (
  model_type = 'BOOSTED_TREE_CLASSIFIER',
  input_label_cols = ['churn'],
  data_split_method = 'NO_SPLIT',
  model_registry = 'VERTEX_AI',          -- registra no Model Registry
  vertex_ai_model_id = 'churn_bqml'
) AS
SELECT * EXCEPT(cliente_id, data_corte) FROM `ml_churn.treino_2026_10`;

SELECT * FROM ML.EVALUATE(MODEL `ml_churn.modelo_bqml`,
  (SELECT * EXCEPT(cliente_id, data_corte) FROM `ml_churn.teste_2026_10`));

SELECT cliente_id, predicted_churn_probs
FROM ML.PREDICT(MODEL `ml_churn.modelo_bqml`,
  (SELECT * FROM `ml_churn.clientes_ativos`));
```

O BigQuery ML é excelente como baseline e, muitas vezes, como solução final para previsões em lote: os dados não saem do BigQuery e o modelo pode ser registrado no Model Registry e implantado em um endpoint.

### 6.7 AutoML

O **Agent Platform AutoML** treina e ajusta modelos automaticamente para dados tabulares, imagens e vídeo a partir de um dataset gerenciado. É útil para obter rapidamente uma referência de desempenho forte; o custo é menor controle sobre o modelo e treinos mais caros que um job customizado simples.

### Laboratório

- [ ] Empacote o treino de churn em um contêiner, publique no Artifact Registry com a tag do commit e rode um custom job.
- [ ] Execute um tuning com 12 trials e compare o melhor resultado com o do notebook.
- [ ] Treine o mesmo problema com BigQuery ML e compare a AUC-PR e o tempo total de desenvolvimento.
- [ ] Rode o custom job em VM spot e verifique, no console, a diferença de custo estimado.

## 7. Model Registry e avaliação

O **Agent Platform Model Registry** (antigo Vertex AI Model Registry) é o catálogo central de modelos: cada modelo tem **versões**, cada versão aponta para seus artefatos, seu contêiner de serving, suas avaliações e sua linhagem. É a partir dele que modelos são implantados.

### 7.1 Versões e aliases

Cada novo treino gera uma **nova versão** do mesmo modelo, em vez de um modelo novo. **Aliases** são rótulos móveis que apontam para uma versão: o alias `default` existe sempre, e você pode criar outros, como `champion` (em produção) e `challenger` (candidato).

```python
# registrar um modelo já treinado como nova versão de um modelo existente
modelo = aiplatform.Model.upload(
    display_name="churn",
    parent_model="projects/ml-dev-123456/locations/southamerica-east1/models/1234567890",
    artifact_uri="gs://ml-dev-123456-ml-artifacts/modelos/churn/2026-11-20/",
    serving_container_image_uri=IMAGEM_SERVING,
    is_default_version=False,
    version_aliases=["challenger"],
    version_description="GB prof=3, dados treino_2026_10, commit a1b2c3d",
    labels={"commit": "a1b2c3d", "dados": "treino_2026_10"},
)

# consultar a versão em produção pelo alias
champion = aiplatform.Model(model_name="1234567890@champion")

# promover: o alias é movido para a nova versão
registro = modelo.versioning_registry
registro.add_version_aliases(new_aliases=["champion"], version=modelo.version_id)
```

### 7.2 Avaliação: o portão de qualidade

Um modelo só deve ser promovido se passar em critérios **objetivos e definidos antes** do treino. Os critérios típicos combinam desempenho absoluto, comparação com o modelo atual e verificações por segmento:

| Critério | Exemplo para churn | Por quê |
| --- | --- | --- |
| Desempenho mínimo | AUC-PR ≥ 0,40 no conjunto de teste | Evita implantar um modelo inútil |
| Não regredir | AUC-PR do challenger ≥ AUC-PR do champion − 0,01, no mesmo conjunto de teste | Evita piorar o que já funciona |
| Segmentos | Nenhuma região ou canal com recall abaixo de 0,30 | Um bom resultado médio pode esconder um grupo mal atendido |
| Calibração | Erro de calibração abaixo de um limite | A operação usa a probabilidade para priorizar ações |
| Operação | Latência p95 abaixo de 100 ms em teste de carga; artefato menor que um limite | O modelo precisa caber no SLA de serving |
| Dados | Validação de dados aprovada para a tabela de treino | Um bom resultado com dados ruins não é confiável |

Os limites acima são ilustrativos: defina-os com o negócio para o seu caso.

```python
def aprovar(challenger: dict, champion: dict | None, limites: dict) -> tuple[bool, list[str]]:
    motivos = []
    if challenger["auc_pr"] < limites["auc_pr_min"]:
        motivos.append(f"AUC-PR {challenger['auc_pr']:.3f} abaixo do mínimo")
    if champion and challenger["auc_pr"] < champion["auc_pr"] - limites["tolerancia"]:
        motivos.append("pior que o champion além da tolerância")
    for segmento, recall in challenger["recall_por_segmento"].items():
        if recall < limites["recall_segmento_min"]:
            motivos.append(f"recall baixo no segmento {segmento}")
    return (not motivos, motivos)
```

### 7.3 Registrando avaliações e explicações

O **Agent Platform Model Evaluation** guarda as métricas de avaliação junto à versão do modelo no Registry, onde podem ser comparadas no console. Em pipelines, há componentes prontos (no pacote `google-cloud-pipeline-components`) que geram avaliações de classificação e regressão a partir de um job de batch inference.

Para **explicabilidade**, configure a **Explainable AI** na versão do modelo (atribuições por Shapley amostrado ou gradientes integrados). Além de exigida em muitos contextos regulados, a importância de features é uma ótima verificação de sanidade: se a feature mais importante não faz sentido para o negócio, suspeite de vazamento.

### 7.4 Fluxo de promoção

1. O pipeline treina e registra a nova versão com o alias `challenger`.
2. A etapa de avaliação compara com o `champion` e aplica os critérios.
3. Aprovado em `ml-dev`/`ml-stg`: o CI/CD copia o modelo para o Registry de produção (ou o pipeline de produção o retreina com os mesmos parâmetros).
4. Em produção, após aprovação (automática ou manual), o alias `champion` é movido para a nova versão e o endpoint é atualizado com tráfego gradual (capítulo 9).
5. A versão anterior fica disponível para **rollback** imediato: basta mover o alias de volta e reimplantar.

### Laboratório

- [ ] Registre o modelo do capítulo 6 como primeira versão com alias `champion`.
- [ ] Treine uma segunda versão com outros hiperparâmetros e registre-a como `challenger` no mesmo modelo.
- [ ] Implemente a função `aprovar` com métricas por segmento e teste-a com casos de aprovação e de reprovação.
- [ ] Promova a versão aprovada movendo o alias e confira o resultado no console do Model Registry.

## 8. Agent Platform Pipelines

O **Agent Platform Pipelines** (antigo Vertex AI Pipelines) executa fluxos de ML definidos com o **Kubeflow Pipelines (KFP) SDK**, sem servidor para gerenciar. Cada etapa roda em seu próprio contêiner, e a plataforma registra automaticamente entradas, saídas, artefatos e linhagem. É a peça que leva você ao nível 1 de maturidade.

&#91;embedded content: Pipeline de treino de churn · 4 etapas e um portão de aprovação\]

O pipeline de churn extrai e valida os dados, treina, avalia e só registra e implanta o modelo se ele passar nos critérios do capítulo 7. Dados inválidos ou um modelo reprovado interrompem o fluxo e geram alerta.

### 8.1 Conceitos

| Conceito | O que é |
| --- | --- |
| Componente | Uma etapa: função Python (*lightweight*) ou contêiner, com entradas e saídas tipadas |
| Pipeline | Um grafo (DAG) de componentes, definido em Python e compilado para YAML |
| Artefato | Saída armazenada no Cloud Storage com metadados: `Dataset`, `Model`, `Metrics`, `ClassificationMetrics` |
| Parâmetro | Valor simples (string, número, booleano) passado entre etapas |
| Execução (*run*) | Uma execução do pipeline compilado, com parâmetros específicos |
| Cache | Etapas com as mesmas entradas e o mesmo código reaproveitam o resultado anterior |
| Google Cloud Pipeline Components | Biblioteca de componentes prontos para BigQuery, treino, batch inference, avaliação e implantação |

### 8.2 Escrevendo os componentes

```python
from kfp import dsl
from kfp.dsl import Input, Output, Dataset, Model, Metrics

BASE = "python:3.12-slim"

@dsl.component(base_image=BASE, packages_to_install=["google-cloud-bigquery", "db-dtypes", "pandas"])
def validar_dados(project: str, tabela: str, min_linhas: int, max_pct_nulos: float) -> bool:
    from google.cloud import bigquery
    r = list(bigquery.Client(project=project).query(f"""
        SELECT COUNT(*) AS n, COUNTIF(dias_cliente IS NULL) / COUNT(*) AS nulos
        FROM `{tabela}`""").result())[0]
    return r.n >= min_linhas and r.nulos <= max_pct_nulos

@dsl.component(base_image=BASE, packages_to_install=["scikit-learn", "pandas", "google-cloud-bigquery", "db-dtypes", "joblib"])
def treinar(project: str, tabela: str, max_depth: int, modelo: Output[Model]):
    import joblib, os
    from google.cloud import bigquery
    from sklearn.ensemble import GradientBoostingClassifier
    feats = ["dias_cliente", "pedidos_90d", "gasto_90d", "dias_desde_ultimo_pedido"]
    df = bigquery.Client(project=project).query(f"SELECT * FROM `{tabela}`").to_dataframe()
    clf = GradientBoostingClassifier(max_depth=max_depth).fit(df[feats], df["churn"])
    os.makedirs(modelo.path, exist_ok=True)
    joblib.dump(clf, os.path.join(modelo.path, "model.joblib"))

@dsl.component(base_image=BASE, packages_to_install=["scikit-learn", "pandas", "google-cloud-bigquery", "db-dtypes", "joblib"])
def avaliar(project: str, tabela_teste: str, auc_pr_min: float,
            modelo: Input[Model], metricas: Output[Metrics]) -> bool:
    import joblib, os
    from google.cloud import bigquery
    from sklearn.metrics import average_precision_score
    feats = ["dias_cliente", "pedidos_90d", "gasto_90d", "dias_desde_ultimo_pedido"]
    df = bigquery.Client(project=project).query(f"SELECT * FROM `{tabela_teste}`").to_dataframe()
    clf = joblib.load(os.path.join(modelo.path, "model.joblib"))
    auc_pr = average_precision_score(df["churn"], clf.predict_proba(df[feats])[:, 1])
    metricas.log_metric("auc_pr", auc_pr)
    return auc_pr >= auc_pr_min
```

O `modelo.path` é um caminho local montado sobre o Cloud Storage: o que você grava ali vira o artefato, sem código de upload.

### 8.3 Montando o pipeline

```python
@dsl.component(base_image=BASE, packages_to_install=["google-cloud-aiplatform"])
def registrar(project: str, location: str, imagem_serving: str,
              modelo: Input[Model], parent_model: str = "") -> str:
    from google.cloud import aiplatform
    aiplatform.init(project=project, location=location)
    versao = aiplatform.Model.upload(
        display_name="churn",
        parent_model=parent_model or None,          # vazio na primeira vez
        artifact_uri=modelo.uri,                    # diretório gs:// do artefato
        serving_container_image_uri=imagem_serving,
        is_default_version=not parent_model,
        version_aliases=["challenger"],
    )
    return versao.resource_name

@dsl.pipeline(name="churn-treino", description="Valida, treina, avalia e registra o modelo de churn")
def pipeline_churn(project: str, location: str, tabela_treino: str, tabela_teste: str,
                   imagem_serving: str, parent_model: str = "",
                   max_depth: int = 3, auc_pr_min: float = 0.40):
    validacao = validar_dados(project=project, tabela=tabela_treino,
                              min_linhas=10_000, max_pct_nulos=0.01)

    with dsl.If(validacao.output == True, name="dados-validos"):
        treino = treinar(project=project, tabela=tabela_treino, max_depth=max_depth)
        treino.set_cpu_limit("4").set_memory_limit("16G")

        avaliacao = avaliar(project=project, tabela_teste=tabela_teste,
                            auc_pr_min=auc_pr_min, modelo=treino.outputs["modelo"])

        with dsl.If(avaliacao.outputs["Output"] == True, name="modelo-aprovado"):
            registrar(project=project, location=location, imagem_serving=imagem_serving,
                      modelo=treino.outputs["modelo"], parent_model=parent_model)
```

Quando um componente tem artefatos de saída **e** um valor de retorno, o retorno fica em `outputs["Output"]`. Para o alerta das ramificações de falha, envolva o pipeline em um `dsl.ExitHandler` com o componente pronto `VertexNotificationEmailOp` (do pacote `google-cloud-pipeline-components`), que envia o status final por e-mail, ou publique em um tópico do Pub/Sub. Os nomes de componentes prontos variam entre versões do KFP e do `google-cloud-pipeline-components`: fixe as versões no `requirements.txt` e confira a documentação delas.

### 8.4 Compilando e executando

```python
from kfp import compiler
from google.cloud import aiplatform

compiler.Compiler().compile(pipeline_churn, "churn_pipeline.yaml")

aiplatform.init(project=PROJECT_ID, location=REGION)
job = aiplatform.PipelineJob(
    display_name="churn-treino",
    template_path="churn_pipeline.yaml",
    pipeline_root=f"gs://{PROJECT_ID}-ml-artifacts/pipelines",
    parameter_values={
        "project": PROJECT_ID,
        "location": REGION,
        "tabela_treino": f"{PROJECT_ID}.ml_churn.treino_2026_10",
        "tabela_teste": f"{PROJECT_ID}.ml_churn.teste_2026_10",
        "imagem_serving": IMAGEM_SERVING,
        "parent_model": "",      # na segunda execução em diante: o resource name do modelo
    },
    enable_caching=True,
    labels={"env": "dev", "model": "churn"},
)
job.submit(service_account=f"sa-pipelines@{PROJECT_ID}.iam.gserviceaccount.com")
```

O YAML compilado é o **artefato** que o CI/CD publica e implanta. Você pode guardá-lo em um repositório **Kubeflow Pipelines** do Artifact Registry, com versionamento e tags, e executá-lo a partir dele.

### 8.5 Cache: economizar sem perder correção

O cache reaproveita etapas cujo código e entradas não mudaram, o que acelera muito o desenvolvimento. Cuidado com etapas que leem dados que **mudam fora do pipeline** (como uma tabela sobrescrita): para o cache, a entrada é o nome da tabela, não o conteúdo. Desative o cache nessas etapas (`tarefa.set_caching_options(False)`) ou passe como parâmetro algo que mude junto com os dados, como a data de corte.

### 8.6 Agendamento e gatilhos

```python
# retreino mensal, no dia 2, às 6h (horário de São Paulo)
job.create_schedule(
    display_name="churn-mensal",
    cron="TZ=America/Sao_Paulo 0 6 2 * *",
    max_concurrent_run_count=1,
    service_account=f"sa-pipelines@{PROJECT_ID}.iam.gserviceaccount.com",
)
```

| Gatilho | Como implementar |
| --- | --- |
| Calendário | Agendamento nativo do pipeline (acima) ou Cloud Scheduler |
| Dados novos | Evento do Cloud Storage ou do BigQuery via Eventarc → Cloud Run function que dispara o pipeline |
| Drift detectado | Alerta do Model Monitoring → Pub/Sub → Cloud Run function (capítulo 11) |
| Mudança de código | CI/CD compila e executa o pipeline (capítulo 10) |

### 8.7 Metadados e linhagem

Toda execução grava no **ML Metadata** quais artefatos foram consumidos e produzidos. No console, a visão de linhagem de um modelo mostra a execução do pipeline que o gerou, os parâmetros, a tabela de dados e as métricas. É isso que responde, meses depois, à pergunta de auditoria: “com quais dados e código este modelo em produção foi treinado?”

### Laboratório

- [ ] Implemente os componentes `validar_dados`, `treinar` e `avaliar` e monte o pipeline com os dois `dsl.If`.
- [ ] Execute o pipeline no ambiente de desenvolvimento e acompanhe o grafo no console.
- [ ] Rode de novo sem mudanças e observe o cache; depois mude `max_depth` e veja quais etapas reexecutam.
- [ ] Forçe uma reprovação (por exemplo, `auc_pr_min=0.99`) e confirme que o registro não acontece.
- [ ] Crie um agendamento mensal e abra a linhagem de um modelo registrado pelo pipeline.

## 9. Implantação e serving

A forma de servir depende de **quando** a previsão é necessária. Se o resultado pode esperar horas, use **batch inference**: é mais simples e muito mais barato. Use **online inference** apenas quando a previsão precisa sair em milissegundos, no meio de uma requisição do usuário.

| Opção | Quando usar | Custo típico |
| --- | --- | --- |
| Agent Platform Batch Inference (antiga Batch Prediction) | Pontuar milhares ou milhões de registros periodicamente | Paga só durante o job |
| BigQuery ML `ML.PREDICT` | Modelo do BigQuery ML ou importado (ONNX, TensorFlow, XGBoost) e dados no BigQuery | Preço de consulta do BigQuery |
| Agent Platform Endpoints (online) | Latência baixa, tráfego contínuo, necessidade de traffic split e monitoramento integrado | Nós ligados 24 h |
| Cloud Run | Modelos leves em contêiner, tráfego irregular, escala a zero | Paga por requisição |
| GKE | Muitos modelos, GPUs compartilhadas, controle fino (KServe, vLLM, Triton) | Cluster gerenciado por você |

### 9.1 Batch inference

Para churn, a operação de retenção normalmente precisa de uma lista diária ou semanal de clientes em risco, então o batch é a escolha certa:

```python
modelo = aiplatform.Model(model_name="1234567890@champion")

batch = modelo.batch_predict(
    job_display_name="churn-pontuacao-semanal",
    bigquery_source=f"bq://{PROJECT_ID}.ml_churn.clientes_ativos",
    bigquery_destination_prefix=f"bq://{PROJECT_ID}.ml_churn_predicoes",
    instances_format="bigquery",
    predictions_format="bigquery",
    machine_type="n2-standard-4",
    starting_replica_count=1,
    max_replica_count=5,
    service_account=f"sa-pipelines@{PROJECT_ID}.iam.gserviceaccount.com",
)
```

O resultado vai para uma tabela do BigQuery, pronta para o CRM consumir. Coloque esse passo em um pipeline agendado separado do pipeline de treino.

### 9.2 Endpoints online

Um **endpoint** é um endereço HTTPS estável; nele você implanta uma ou mais versões de modelos e divide o tráfego entre elas.

```python
endpoint = aiplatform.Endpoint.create(display_name="churn-endpoint", labels={"model": "churn"})

modelo.deploy(
    endpoint=endpoint,
    deployed_model_display_name="churn-v3",
    machine_type="n2-standard-4",
    min_replica_count=1,
    max_replica_count=4,                 # autoscaling por uso de CPU
    autoscaling_target_cpu_utilization=60,
    traffic_percentage=100,
    service_account=f"sa-endpoint@{PROJECT_ID}.iam.gserviceaccount.com",
    enable_access_logging=True,
)

resposta = endpoint.predict(instances=[
    {"dias_cliente": 420, "pedidos_90d": 1, "gasto_90d": 89.9, "dias_desde_ultimo_pedido": 75}
])
print(resposta.predictions)
```

Para chamadas de sistemas internos sem sair da rede, crie um **endpoint privado** (via Private Service Connect). Para tráfego alto e baixa latência, avalie os **endpoints dedicados**, que têm DNS próprio e isolamento de tráfego.

### 9.3 Implantação gradual: canary e rollback

Nunca troque 100% do tráfego de uma vez. Implante a nova versão no mesmo endpoint com uma fatia pequena e aumente aos poucos, observando latência, erros e distribuição das previsões:

```python
nova = aiplatform.Model(model_name="1234567890@challenger")
nova.deploy(endpoint=endpoint, machine_type="n2-standard-4",
            min_replica_count=1, max_replica_count=4, traffic_percentage=10)

# depois de validar: 50% e, por fim, 100%
ids = {m.display_name: m.id for m in endpoint.list_models()}
endpoint.update(traffic_split={ids["churn-v4"]: 100, ids["churn-v3"]: 0})

# rollback imediato, se necessário
endpoint.update(traffic_split={ids["churn-v4"]: 0, ids["churn-v3"]: 100})
```

Mantenha a versão anterior implantada com 0% de tráfego por alguns dias para um rollback instantâneo e depois remova-a com `endpoint.undeploy(...)`, porque ela continua consumindo nós.

| Estratégia | Como funciona | Quando usar |
| --- | --- | --- |
| Canary | Fatia crescente do tráfego real vai para a nova versão | Padrão para a maioria das trocas |
| Shadow | A nova versão recebe cópia do tráfego, mas sua resposta não é usada | Quando um erro em produção é caro; exige código na aplicação chamadora |
| Teste A/B | Grupos de usuários recebem versões diferentes e o impacto no negócio é medido | Quando a métrica que importa é de negócio, não só offline |

### 9.4 Contêiner de serving customizado

Se os contêineres pré-construídos não atenderem (pré-processamento próprio, vários artefatos, outro framework), crie o seu. O contrato é simples: um servidor HTTP que responde em `AIP_PREDICT_ROUTE` e `AIP_HEALTH_ROUTE`, na porta `AIP_HTTP_PORT`, e carrega o modelo de `AIP_STORAGE_URI`:

```python
# app.py — FastAPI
import os, joblib, pandas as pd
from fastapi import FastAPI, Request
from google.cloud import storage

FEATURES = ["dias_cliente", "pedidos_90d", "gasto_90d", "dias_desde_ultimo_pedido"]
app = FastAPI()

bucket, prefixo = os.environ["AIP_STORAGE_URI"].removeprefix("gs://").split("/", 1)
storage.Client().bucket(bucket).blob(f"{prefixo}/model.joblib").download_to_filename("model.joblib")
modelo = joblib.load("model.joblib")

@app.get(os.environ.get("AIP_HEALTH_ROUTE", "/health"))
def saude():
    return {"status": "ok"}

@app.post(os.environ.get("AIP_PREDICT_ROUTE", "/predict"))
async def prever(req: Request):
    instancias = (await req.json())["instances"]
    prob = modelo.predict_proba(pd.DataFrame(instancias)[FEATURES])[:, 1]
    return {"predictions": [{"prob_churn": float(p)} for p in prob]}
```

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir fastapi uvicorn scikit-learn pandas joblib google-cloud-storage
COPY app.py .
CMD exec uvicorn app:app --host 0.0.0.0 --port ${AIP_HTTP_PORT:-8080}
```

Mantenha o pré-processamento **dentro** do artefato ou do contêiner de serving e reutilize exatamente o mesmo código do treino. Duas implementações diferentes do mesmo cálculo são a origem clássica do *training-serving skew*.

### 9.5 Cloud Run e GKE

O mesmo contêiner roda no **Cloud Run** com `gcloud run deploy churn --image=$IMAGEM_SERVING --region=$REGION --no-allow-unauthenticated`, o que é ótimo para modelos pequenos com tráfego irregular (escala a zero). Você perde os recursos integrados dos endpoints, como traffic split por versão de modelo e Model Monitoring; o Cloud Run tem divisão de tráfego por revisão, e o monitoramento passa a ser feito a partir dos logs. No **GKE**, use KServe, NVIDIA Triton ou vLLM quando precisar servir muitos modelos ou LLMs abertos com GPUs compartilhadas.

### Laboratório

- [ ] Rode um batch inference do `champion` sobre os clientes ativos, gravando no BigQuery.
- [ ] Crie um endpoint, implante o `champion`, faça previsões com o SDK e com `curl` usando `gcloud auth print-access-token`.
- [ ] Implante o `challenger` com 10% do tráfego, depois 100%, e faça um rollback.
- [ ] Construa o contêiner de serving customizado, teste-o localmente com Docker e implante-o também no Cloud Run.
- [ ] Remova as implantações do endpoint ao final para não pagar por nós ociosos.

## 10. CI/CD para ML

No nível 2 de maturidade, nenhuma pessoa implanta nada à mão em homologação ou produção. Um commit dispara testes, constrói imagens, compila o pipeline e o implanta em cada ambiente, com aprovação explícita antes da produção.

### 10.1 O que o CI/CD entrega

| Artefato | Onde fica | Versionado por |
| --- | --- | --- |
| Imagens de treino e serving | Artifact Registry (Docker) | Hash do commit |
| Pipeline compilado (`.yaml`) | Artifact Registry (Kubeflow Pipelines) ou Cloud Storage | Hash do commit e tag de versão |
| Infraestrutura | Terraform | Commit do repositório de infraestrutura |
| Modelo | Model Registry | Versão e aliases, com o commit nos labels |

### 10.2 Os estágios

1. **Pull request:** lint, testes unitários dos componentes e das funções de features, teste de que o pipeline compila.
2. **Merge na branch principal:** constrói e publica imagens, compila e publica o pipeline, executa o pipeline em `ml-stg` com uma amostra dos dados (teste de integração).
3. **Release:** com aprovação manual, publica o mesmo pipeline (mesmo artefato, não recompilado) em `ml-prd` e atualiza o agendamento.
4. **Promoção de modelo:** o pipeline de produção treina, avalia e registra; o deploy gradual no endpoint é disparado quando o modelo é aprovado.

> **Princípio:** o mesmo artefato percorre todos os ambientes. Recompilar ou reconstruir a imagem para produção significa implantar algo que não foi testado.

### 10.3 Testes que valem a pena

| Teste | Exemplo |
| --- | --- |
| Unitário de features | A função de `dias_desde_ultimo_pedido` devolve o valor esperado para um cliente sem pedidos |
| Unitário de critério de aprovação | `aprovar()` reprova um modelo abaixo do mínimo |
| Compilação | `compiler.Compiler().compile(...)` roda sem erro no CI |
| Contrato do serving | O contêiner sobe, responde ao health check e devolve o formato esperado para uma instância de exemplo |
| Integração | O pipeline completo executa em `ml-stg` com 1% dos dados em poucos minutos |
| Infraestrutura | `terraform plan` sem mudanças inesperadas |

### 10.4 GitHub Actions com Workload Identity Federation

O **Workload Identity Federation** permite que o GitHub Actions se autentique na Google Cloud sem chave JSON: o GitHub emite um token OIDC de curta duração que a Google Cloud troca por credenciais da `sa-cicd`.

```bash
gcloud iam workload-identity-pools create github --location=global
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global --workload-identity-pool=github \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='minha-org/churn-mlops'"

gcloud iam service-accounts add-iam-policy-binding sa-cicd@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/minha-org/churn-mlops"
```

```yaml
# .github/workflows/ml.yaml
name: churn-mlops
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write          # necessário para o token OIDC

env:
  REGION: southamerica-east1
  PROJECT_ID: ml-stg-123456

jobs:
  testes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements-dev.txt
      - run: ruff check . && pytest -q
      - run: python -m pipelines.compilar --saida churn_pipeline.yaml

  publicar-e-testar:
    needs: testes
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github/providers/github-provider
          service_account: sa-cicd@ml-stg-123456.iam.gserviceaccount.com
      - uses: google-github-actions/setup-gcloud@v2
      - name: Construir e publicar imagens
        run: |
          TAG=${GITHUB_SHA::7}
          gcloud builds submit treino --tag $REGION-docker.pkg.dev/$PROJECT_ID/ml-images/churn-trainer:$TAG
          gcloud builds submit serving --tag $REGION-docker.pkg.dev/$PROJECT_ID/ml-images/churn-serving:$TAG
      - name: Executar pipeline em homologação
        run: python -m pipelines.executar --ambiente stg --tag ${GITHUB_SHA::7} --amostra 0.01 --aguardar
```

O job de produção fica em um *workflow* separado, disparado por tag de release, usando um **GitHub Environment** com revisores obrigatórios e outra service account, que só tem acesso a `ml-prd`.

### 10.5 Alternativa: Cloud Build

Se o código estiver em um repositório conectado ao Cloud Build, o mesmo fluxo vira um `cloudbuild.yaml` disparado por gatilhos de push ou de tag:

```yaml
steps:
  - name: python:3.12
    entrypoint: bash
    args: ["-c", "pip install -r requirements-dev.txt && pytest -q"]
  - name: gcr.io/cloud-builders/docker
    args: ["build", "-t", "$_REGION-docker.pkg.dev/$PROJECT_ID/ml-images/churn-trainer:$SHORT_SHA", "treino"]
  - name: python:3.12
    entrypoint: bash
    args: ["-c", "pip install -r requirements.txt && python -m pipelines.executar --ambiente stg --tag $SHORT_SHA --aguardar"]
images:
  - $_REGION-docker.pkg.dev/$PROJECT_ID/ml-images/churn-trainer:$SHORT_SHA
serviceAccount: projects/$PROJECT_ID/serviceAccounts/sa-cicd@$PROJECT_ID.iam.gserviceaccount.com
options:
  logging: CLOUD_LOGGING_ONLY
substitutions:
  _REGION: southamerica-east1
```

### 10.6 Estrutura de repositório sugerida

```text
churn-mlops/
├── churn/              # pacote Python: dados, features, treino, avaliação (com testes)
├── treino/             # Dockerfile da imagem de treino
├── serving/            # Dockerfile e app.py do serving
├── pipelines/          # componentes KFP, definição do pipeline, scripts compilar/executar
├── config/             # parâmetros por ambiente (dev.yaml, stg.yaml, prd.yaml)
├── infra/              # Terraform
├── tests/
└── .github/workflows/
```

### Laboratório

- [ ] Organize o código dos capítulos anteriores na estrutura acima, com parâmetros por ambiente em `config/`.
- [ ] Configure o Workload Identity Federation para o seu repositório e confirme que não há nenhuma chave de service account.
- [ ] Crie o workflow de PR com testes e compilação, e o de merge que publica imagens e executa o pipeline em homologação.
- [ ] Crie o workflow de produção com aprovação manual e verifique que ele reutiliza o artefato já testado.

## 11. Monitoramento e retreino

Um modelo em produção degrada em silêncio: o endpoint continua respondendo 200 OK enquanto as previsões pioram. Por isso o monitoramento de ML tem três camadas — **operação** (o serviço está de pé?), **dados** (as entradas mudaram?) e **desempenho** (as previsões ainda acertam?).

### 11.1 O que pode mudar

| Fenômeno | O que é | Exemplo no churn |
| --- | --- | --- |
| *Training-serving skew* | Os dados de produção são diferentes dos de treino desde o início | Em produção, `gasto_90d` é calculado em reais com impostos; no treino, sem impostos |
| *Data drift* (*prediction drift*) | A distribuição das entradas muda ao longo do tempo | Uma campanha traz muitos clientes novos, e `dias_cliente` cai |
| *Concept drift* | A relação entre entradas e rótulo muda | Um concorrente lança um plano mais barato, e clientes ativos passam a sair |
| Problemas de dados | Nulos, valores fora de faixa, schema alterado | Uma integração passa a enviar `pedidos_90d` vazio |

Skew e drift são detectáveis **sem rótulos**, comparando distribuições. Concept drift só aparece quando os rótulos reais chegam, o que no churn leva 60 dias.

### 11.2 Model Monitoring on Agent Platform

O **Model Monitoring** (antigo Vertex AI Model Monitoring) compara as entradas de produção com uma referência (os dados de treino, para skew; uma janela anterior, para drift), calcula a distância entre as distribuições de cada feature e alerta quando um limite é ultrapassado. Existem duas versões ([visão geral oficial](https://docs.cloud.google.com/vertex-ai/docs/model-monitoring/overview)):

|  | Model Monitoring v1 | Model Monitoring v2 |
| --- | --- | --- |
| Status | Geralmente disponível (GA) | *Preview* |
| Configurado em | Endpoint | Versão do modelo no Model Registry |
| Fontes monitoradas | Tráfego do endpoint | Endpoints, batch inference e modelos servidos fora da plataforma |
| Tipos de modelo | Tabulares | Somente tabulares |
| Recomendação oficial | Produção com suporte em endpoint da plataforma | Demais casos |

Para o endpoint de churn, um job v1 com skew e drift fica assim:

```python
from google.cloud.aiplatform import model_monitoring

skew = model_monitoring.SkewDetectionConfig(
    data_source=f"bq://{PROJECT_ID}.ml_churn.treino_2026_10",
    target_field="churn",
    skew_thresholds={"gasto_90d": 0.3, "dias_cliente": 0.3, "pedidos_90d": 0.3},
)
drift = model_monitoring.DriftDetectionConfig(
    drift_thresholds={"gasto_90d": 0.3, "dias_cliente": 0.3, "pedidos_90d": 0.3},
)

job = aiplatform.ModelDeploymentMonitoringJob.create(
    display_name="churn-monitoramento",
    endpoint=endpoint,
    logging_sampling_strategy=model_monitoring.RandomSampleConfig(sample_rate=0.5),
    schedule_config=model_monitoring.ScheduleConfig(monitor_interval=6),   # horas
    alert_config=model_monitoring.EmailAlertConfig(
        user_emails=["ml-oncall@empresa.com.br"], enable_logging=True),
    objective_configs=model_monitoring.ObjectiveConfig(skew, drift),
)
```

Com `enable_logging=True`, os alertas também vão para o Cloud Logging, de onde podem disparar automações. Os limites de 0,3 são um ponto de partida: ajuste-os observando o comportamento normal das features por algumas semanas, para evitar alertas que ninguém lê.

### 11.3 Monitorando desempenho com rótulos reais

Grave toda previsão (com id do cliente, versão do modelo e data) no BigQuery: o *request-response logging* do endpoint ou as tabelas de saída do batch inference fazem isso. Quando os rótulos amadurecem, calcule o desempenho real em uma consulta agendada:

```sql
CREATE OR REPLACE VIEW `ml_churn.desempenho_mensal` AS
SELECT
  DATE_TRUNC(p.data_predicao, MONTH) AS mes,
  p.versao_modelo,
  COUNT(*) AS clientes,
  AVG(CAST(r.churn AS INT64)) AS taxa_churn_real,
  -- precisão entre os 10% de maior risco, a lista que a operação recebe
  AVG(IF(p.decil_risco = 10, CAST(r.churn AS INT64), NULL)) AS precisao_top10
FROM `ml_churn_predicoes.historico` p
JOIN `ml_churn.rotulos_reais` r
  USING (cliente_id, data_corte)
GROUP BY mes, versao_modelo;
```

Acompanhe essa métrica em um dashboard e crie um alerta se ela cair abaixo do esperado.

### 11.4 Monitoramento operacional

No **Cloud Monitoring**, os endpoints publicam métricas de requisições, latência, erros e uso de CPU, memória e acelerador. Configure pelo menos:

- alerta de latência p95 acima do SLA;
- alerta de taxa de erros 5xx acima de 1%;
- alerta de réplicas no máximo do autoscaling por mais de 15 minutos;
- alerta de falha em execuções de pipelines e de batch inference (por métricas baseadas em logs);
- um dashboard único com operação, drift e desempenho por versão do modelo.

### 11.5 Fechando o ciclo: retreino automático

1. O Model Monitoring detecta drift e grava o alerta no Cloud Logging.
2. Um **coletor de logs** (*log sink*) envia esses alertas para um tópico do Pub/Sub.
3. Uma **Cloud Run function** inscrita no tópico dispara o pipeline de treino com dados atualizados.
4. O pipeline valida, treina, avalia e, se aprovado, registra o novo `challenger`.
5. O deploy gradual segue o fluxo do capítulo 9, com aprovação humana se a política exigir.

```python
# main.py — Cloud Run function acionada pelo Pub/Sub
import functions_framework
from datetime import date
from google.cloud import aiplatform

@functions_framework.cloud_event
def retreinar(evento):
    aiplatform.init(project="ml-prd-123456", location="southamerica-east1")
    aiplatform.PipelineJob(
        display_name=f"churn-retreino-{date.today():%Y%m%d}",
        template_path="https://southamerica-east1-kfp.pkg.dev/ml-prd-123456/pipelines/churn-treino/v1.4.0",
        pipeline_root="gs://ml-prd-123456-ml-artifacts/pipelines",
        parameter_values={"data_corte": date.today().isoformat()},   # versão de produção: gera as tabelas a partir da data de corte
    ).submit(service_account="sa-pipelines@ml-prd-123456.iam.gserviceaccount.com")
```

> **Atenção:** retreinar não corrige tudo. Se o alerta for causado por um bug de dados (uma integração quebrada), retreinar ensina o modelo a aceitar o erro. Por isso a validação de dados do pipeline vem antes do treino, e por isso alertas de drift grandes e repentinos devem ir para uma pessoa antes do retreino.

### Laboratório

- [ ] Ative o monitoramento de skew e drift no endpoint de churn com amostragem de 50%.
- [ ] Envie ao endpoint um lote de requisições com `gasto_90d` multiplicado por 3 e observe o alerta.
- [ ] Crie a view de desempenho mensal com rótulos simulados e um alerta no Cloud Monitoring.
- [ ] Implemente a cadeia alerta → Pub/Sub → Cloud Run function → pipeline e teste-a publicando uma mensagem manualmente no tópico.

## 12. GenAI e LLMOps

Com modelos fundacionais, o ciclo de MLOps muda de forma: em vez de treinar, você **escolhe um modelo, escreve prompts, conecta dados (RAG) e, às vezes, faz tuning**. Os princípios continuam os mesmos — versionar, avaliar antes de promover, monitorar —, mas os artefatos passam a ser prompts, configurações, índices de busca e conjuntos de avaliação. Essa disciplina é chamada de **LLMOps**.

### 12.1 O que muda em relação ao ML clássico

| Aspecto | ML clássico | LLMOps |
| --- | --- | --- |
| Artefato principal | Pesos do modelo treinado | Prompt, versão do modelo, configuração de RAG e de ferramentas |
| Avaliação | Métricas sobre um conjunto rotulado | Conjunto de avaliação com respostas de referência, critérios avaliados por outro modelo (*LLM-as-judge*) e revisão humana |
| Mudança de versão | Você retreina | O provedor lança e aposenta versões do modelo |
| Custo | Infraestrutura de treino e serving | Tokens de entrada e saída, por requisição |
| Riscos novos | — | Alucinação, *prompt injection*, vazamento de dados pelo contexto |

### 12.2 Gemini e Model Garden

O **Gemini** está disponível na Agent Platform, e o **Model Garden** oferece também modelos abertos (como Gemma) e de terceiros. As famílias e versões mudam com frequência; em novembro de 2026, as versões Gemini 2.5 já tinham data de aposentadoria em 16 de outubro de 2026, e a série Gemini 3 era a atual ([versões dos modelos](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions)). Por isso, **o ID do modelo é configuração**, nunca uma constante espalhada pelo código.

Use o **Google Gen AI SDK** (`google-genai`), o SDK atual para Gemini:

```python
from google import genai
from google.genai import types

client = genai.Client(vertexai=True, project=PROJECT_ID, location="global")
MODELO = config["modelo"]    # por exemplo, uma versão estável da série Gemini 3, vinda do config/prd.yaml

resposta = client.models.generate_content(
    model=MODELO,
    contents="Resuma em até 3 frases o motivo do contato deste cliente: ...",
    config=types.GenerateContentConfig(
        system_instruction="Você é um analista de atendimento. Responda em português.",
        temperature=0.2,
        max_output_tokens=300,
    ),
)
print(resposta.text)
```

> **Versão fixa ou alias?** O alias atualizado automaticamente de um modelo sempre aponta para a versão estável mais recente, e muda sozinho quando sai uma nova ([documentação](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions)). Em produção, **fixe a versão** e faça a migração como qualquer outra mudança: rode a avaliação com a versão nova e só promova se ela passar. Acompanhe as datas de aposentadoria para não ser pego de surpresa.

### 12.3 Prompts como código

Guarde prompts em arquivos versionados no Git (por exemplo, `prompts/resumo_contato/v3.yaml`), com o modelo, os parâmetros e o *system instruction*. Cada mudança de prompt passa por PR e pela avaliação automatizada, exatamente como uma mudança de código. O **Agent Studio** (antigo Vertex AI Studio) é ótimo para prototipar e salvar prompts, mas a versão que vai para produção deve estar no repositório.

### 12.4 Avaliação com Gen AI evals

O **Gen AI evals** (antigo Gen AI evaluation service) avalia respostas com métricas baseadas em cálculo (correspondência exata, ROUGE, BLEU) e métricas baseadas em modelo, em que um modelo juiz pontua critérios como fidelidade ao contexto, relevância, segurança e aderência a instruções, ou critérios que você define.

Monte um **conjunto de avaliação** com 50 a 200 casos reais e representativos (incluindo casos difíceis e tentativas de abuso), com respostas de referência ou critérios de aceitação. No CI/CD, toda mudança de prompt, de modelo ou de RAG roda essa avaliação e é comparada com a versão atual, com o mesmo raciocínio de portão de qualidade do capítulo 7. A API do serviço evolui rápido: siga o guia de início rápido da documentação atual para o código exato.

### 12.5 RAG: respondendo com os seus dados

**RAG** (*Retrieval-Augmented Generation*) busca trechos relevantes nos seus documentos e os coloca no contexto do prompt, para que o modelo responda com base neles. As opções na plataforma:

| Opção | Quando usar |
| --- | --- |
| **RAG Engine** | Pipeline de RAG gerenciado: ingestão, divisão em trechos, embeddings, índice e recuperação. Tem modo *serverless* (em preview) |
| **Vector Search** | Índice vetorial de alta escala quando você controla os embeddings e a lógica de recuperação |
| **Agent Search** (antigo Vertex AI Search) | Busca empresarial pronta sobre sites, documentos e dados estruturados |
| **BigQuery** com funções vetoriais | Dados e embeddings já no BigQuery, busca integrada ao SQL |
| **Grounding com Google Search** | Respostas que dependem de informação pública e atual |

Monitore em RAG duas coisas separadamente: a **qualidade da recuperação** (os trechos certos foram encontrados?) e a **qualidade da geração** (a resposta é fiel aos trechos?). A maioria dos problemas está na recuperação.

### 12.6 Tuning

Antes de fazer tuning, esgote prompt e RAG: são mais baratos e mais fáceis de manter. O **tuning supervisionado** do Gemini vale a pena para padronizar formato e estilo, ou para tarefas muito específicas com centenas a milhares de exemplos de alta qualidade. Para modelos abertos (Gemma, Llama), use o Managed Training com LoRA e sirva no endpoint ou no GKE. O modelo ajustado vai para o Model Registry e segue o mesmo fluxo de avaliação e promoção.

### 12.7 Agentes

Agentes combinam um modelo com ferramentas (APIs, consultas, busca) e memória. Na plataforma, você pode construí-los com o **Agent Development Kit (ADK)** ou outros frameworks e implantá-los no **Agent Runtime** (antigo Vertex AI Agent Engine), que oferece sessões, memória e execução de código gerenciadas. Para LLMOps de agentes, registre o *trace* de cada passo (chamadas ao modelo e às ferramentas) e avalie também a **trajetória**, não apenas a resposta final.

### 12.8 Operação e governança de GenAI

- **Custo:** registre tokens de entrada e saída por requisição e por funcionalidade; use *context caching* para contextos longos repetidos e a API em lote para cargas sem urgência.
- **Latência e capacidade:** use streaming nas interfaces e avalie **Provisioned Throughput** para tráfego alto e previsível.
- **Segurança:** configure os filtros de segurança, trate a entrada do usuário como não confiável (*prompt injection*), nunca coloque segredos no prompt e limite as permissões das ferramentas dos agentes.
- **Dados:** verifique os termos de uso e retenção de dados do serviço e mascare dados pessoais antes de enviá-los ao modelo quando não forem necessários.
- **Monitoramento:** amostre respostas de produção e rode a mesma avaliação periodicamente; colete feedback dos usuários.

### Laboratório

- [ ] Crie um prompt que resuma o motivo de contato de clientes em risco de churn, com o ID do modelo vindo da configuração.
- [ ] Monte um conjunto de avaliação com 30 casos e avalie duas versões do prompt com o Gen AI evals.
- [ ] Crie um corpus no RAG Engine com as políticas de retenção da empresa e responda perguntas com base nele.
- [ ] Adicione ao CI/CD um passo que bloqueie o merge se a nota média da avaliação cair.

## 13. Segurança, governança e custos

Um sistema de ML lida com dados sensíveis e toma decisões que afetam pessoas. Segurança e governança não são uma etapa final: são restrições que valem para todos os capítulos anteriores. Este capítulo reúne os controles em uma lista que você pode usar como revisão.

### 13.1 Segurança da plataforma

| Controle | O que protege | Como aplicar |
| --- | --- | --- |
| Service accounts dedicadas e papéis mínimos | Acesso indevido a dados e recursos | Capítulo 3; revise com o IAM Recommender |
| Sem chaves de service account | Vazamento de credenciais | Política `iam.disableServiceAccountKeyCreation`; impersonation e Workload Identity Federation |
| VPC Service Controls | Exfiltração de dados para fora da organização | Perímetro com projetos de dados e ML e os serviços `aiplatform`, `bigquery` e `storage` restritos |
| Rede privada | Exposição de endpoints e jobs à internet | Endpoints privados, Private Service Connect, jobs com acesso à VPC |
| CMEK | Exigências de controle sobre chaves de criptografia | Chaves do Cloud KMS em buckets, datasets, pipelines, jobs e endpoints |
| Secret Manager | Senhas e tokens em código ou imagens | Leitura em tempo de execução pela service account |
| Artifact Analysis | Vulnerabilidades nas imagens | Varredura automática no Artifact Registry; bloqueie imagens críticas no CI |
| Cloud Audit Logs | Investigação e auditoria | Logs de acesso a dados habilitados para BigQuery e para a Agent Platform |
| Políticas da organização | Configurações inseguras ou fora da região | Restrição de locais de recursos (por exemplo, apenas regiões no Brasil) e de IPs públicos |

### 13.2 Dados pessoais e LGPD

- **Minimização:** use apenas as features necessárias; identificadores diretos (nome, CPF, e-mail) quase nunca são features úteis.
- **Pseudonimização:** substitua identificadores por chaves com hash antes de levar dados ao ambiente de ML. O **Sensitive Data Protection** (antigo Cloud DLP) encontra e mascara dados pessoais em tabelas e textos.
- **Controle de acesso fino:** políticas de linha e tags de política de coluna no BigQuery restringem quem vê dados sensíveis.
- **Residência:** mantenha dados e processamento nas regiões exigidas pela sua política.
- **Retenção:** defina expiração para tabelas de treino, logs de previsão e artefatos antigos.
- **Base legal e transparência:** decisões automatizadas que afetam pessoas podem exigir explicação e revisão. Envolva as áreas jurídica e de privacidade antes de usar o modelo.

Este guia não substitui orientação jurídica: valide os requisitos específicos com os responsáveis pela privacidade na sua organização.

### 13.3 IA responsável

- **Viés e equidade:** avalie métricas por grupo (capítulo 7) e investigue diferenças relevantes antes de implantar.
- **Explicabilidade:** use a Explainable AI para entender quais features pesam nas previsões e para justificar decisões quando necessário.
- **Documentação:** mantenha um **model card** por modelo, com objetivo, dados usados, métricas por segmento, limitações conhecidas e responsável.
- **Humano no circuito:** para decisões de alto impacto, o modelo recomenda e uma pessoa decide.
- **GenAI:** filtros de segurança, testes de *red teaming* e avaliação de alucinação no conjunto de avaliação.

### 13.4 Governança de modelos

Governança é conseguir responder, a qualquer momento, às perguntas: **quais modelos estão em produção, quem é o dono de cada um, com quais dados foram treinados, quando foram avaliados e quem aprovou?** As peças que respondem a isso já foram construídas ao longo do guia: Model Registry com labels de dono e commit, linhagem do ML Metadata, avaliações registradas, aprovações no CI/CD e logs de auditoria. Reúna-as em um painel ou relatório periódico.

### 13.5 Otimização de custos

| Onde o custo aparece | Como reduzir |
| --- | --- |
| Endpoints online ociosos | Use batch quando possível; `min_replica_count` adequado; remova implantações antigas; considere Cloud Run com escala a zero |
| Treinos com GPU | VMs spot com checkpoints; máquina do tamanho certo; teste com amostras antes de rodar completo |
| Pipelines | Cache habilitado; etapas leves em máquinas pequenas; não retreinar sem necessidade |
| Notebooks | Desligamento por inatividade; Colab Enterprise para uso esporádico |
| BigQuery | Tabelas particionadas e clusterizadas; selecione só as colunas necessárias; reservas de slots se o uso for alto e previsível |
| Armazenamento | Regras de ciclo de vida no Cloud Storage para artefatos de pipelines antigos |
| GenAI | Modelo menor quando a qualidade permite; context caching; limites de tokens de saída; API em lote |

Com labels padronizados e a exportação de faturamento para o BigQuery (capítulo 3), você consegue um relatório de **custo por modelo**, que costuma mudar a conversa sobre quais modelos valem a pena manter.

### Laboratório

- [ ] Revise as permissões das service accounts com o IAM Recommender e remova as não usadas.
- [ ] Pseudonimize o `cliente_id` no conjunto de treino com hash e chave secreta guardada no Secret Manager.
- [ ] Escreva o model card do modelo de churn.
- [ ] Monte uma consulta de custo por modelo a partir da exportação de faturamento e dos labels.

## 14. Projeto final, checklist de produção e trilha de estudos

O projeto final junta os laboratórios em um sistema completo de previsão de churn, do commit ao retreino automático. Ele é o seu portfólio de MLOps: um repositório que outra pessoa consegue ler, implantar e auditar.

### 14.1 Entregas do projeto

| # | Entrega | Critério de pronto | Capítulos |
| --- | --- | --- | --- |
| 1 | Fundação | Projetos `dev` e `prd` (ou simulados), APIs, service accounts e bucket criados só por Terraform | 3 |
| 2 | Dados | Consulta de features com data de corte, sem vazamento; validações que interrompem o pipeline | 4 |
| 3 | Experimentos | Baseline e candidatas registradas no Experiments, com a escolha justificada | 5 |
| 4 | Treino | Imagem versionada pelo commit; custom job e tuning reproduzíveis | 6 |
| 5 | Registro | Versões com aliases `champion` e `challenger`, e função de aprovação testada | 7 |
| 6 | Pipeline | Pipeline KFP com validação, treino, avaliação e registro condicional; agendamento mensal | 8 |
| 7 | Serving | Batch inference semanal no BigQuery e endpoint online com deploy canary e rollback testado | 9 |
| 8 | CI/CD | PR com testes; merge publica e testa em homologação; release com aprovação para produção; sem chaves de service account | 10 |
| 9 | Monitoramento | Skew e drift com alertas; view de desempenho real; dashboard operacional | 11 |
| 10 | Retreino | Cadeia alerta → Pub/Sub → Cloud Run function → pipeline funcionando | 11 |
| 11 | GenAI (opcional) | Resumo de motivos de contato para clientes em risco, com prompt versionado e avaliação no CI | 12 |
| 12 | Governança | Model card, relatório de custo por modelo e README com arquitetura e instruções | 13 |

### 14.2 Checklist de produção

Use esta lista antes de colocar qualquer modelo em produção.

**Dados**

- [ ] Features calculadas sem vazamento e com o mesmo código no treino e no serving
- [ ] Validação de dados automática antes de cada treino
- [ ] Conjunto de treino versionado e rastreável
- [ ] Dados pessoais minimizados e protegidos

**Modelo**

- [ ] Métrica principal acordada com o negócio e comparada com um baseline
- [ ] Métricas por segmento revisadas
- [ ] Critérios de aprovação automatizados no pipeline
- [ ] Model card escrito

**Pipeline e entrega**

- [ ] Treino 100% reproduzível por pipeline, sem passos manuais
- [ ] Imagens e pipeline versionados pelo commit, o mesmo artefato em todos os ambientes
- [ ] Testes unitários, de contrato do serving e de integração no CI
- [ ] Aprovação explícita para produção

**Serving**

- [ ] Modo de serving adequado (batch quando possível)
- [ ] Autoscaling e limites configurados e testados com carga
- [ ] Deploy gradual e procedimento de rollback testados

**Operação**

- [ ] Alertas de latência, erros, falhas de pipeline, skew e drift
- [ ] Desempenho real medido quando os rótulos chegam
- [ ] Dono do modelo e plantão definidos, com runbook para os alertas
- [ ] Orçamento, labels e relatório de custo

**Segurança**

- [ ] Service accounts dedicadas com papéis mínimos e sem chaves
- [ ] Rede privada e VPC Service Controls onde aplicável
- [ ] Logs de auditoria habilitados

### 14.3 Trilha de estudos

1. **Fundamentos de ML:** validação, métricas, vazamento, desbalanceamento de classes, calibração.
2. **Engenharia:** Python de produção (pacotes, testes, tipagem), Docker, Git, Terraform.
3. **Dados na Google Cloud:** BigQuery avançado, Dataflow, Dataplex.
4. **Plataforma:** documentação e codelabs da Agent Platform, sobretudo Pipelines, Model Registry e Model Monitoring.
5. **Leituras de referência:** o documento do Google *MLOps: Continuous delivery and automation pipelines in machine learning* (base dos níveis de maturidade deste guia) e o artigo *Hidden Technical Debt in Machine Learning Systems*.
6. **GenAI:** RAG, avaliação de LLMs e agentes com o ADK.
7. **Kubernetes:** GKE para cenários de serving avançado.

### 14.4 Certificação: Professional Machine Learning Engineer

A certificação do Google Cloud mais alinhada a este guia é a **Professional Machine Learning Engineer**. O exame dura duas horas, custa US$ 200 (mais impostos) e é oferecido em inglês e japonês ([página da certificação](https://cloud.google.com/learn/certification/machine-learning-engineer)). O guia oficial do exame foi atualizado em 1º de junho de 2026, e a prova foi ajustada para os novos nomes de produtos ([guia do exame](https://services.google.com/fh/files/misc/professional_machine_learning_engineer_exam_guide_english_new.pdf)). O Google recomenda mais de três anos de experiência prática com produtos da Google Cloud antes da prova.

As áreas avaliadas incluem arquitetar soluções de IA com pouco código, colaborar entre equipes para gerenciar dados e modelos, escalar protótipos para modelos de ML, servir e escalar modelos, automatizar pipelines e monitorar soluções. Cada capítulo deste guia corresponde a uma dessas áreas; use o guia oficial do exame como checklist final de revisão.

## 15. Apêndices

### A. Comandos `gcloud` mais usados

| Tarefa | Comando |
| --- | --- |
| Habilitar a API da plataforma | `gcloud services enable aiplatform.googleapis.com` |
| Listar modelos do Registry | `gcloud ai models list --region=$REGION` |
| Ver versões de um modelo | `gcloud ai models list-version MODEL_ID --region=$REGION` |
| Listar endpoints | `gcloud ai endpoints list --region=$REGION` |
| Fazer uma previsão | `gcloud ai endpoints predict ENDPOINT_ID --region=$REGION --json-request=req.json` |
| Remover um modelo de um endpoint | `gcloud ai endpoints undeploy-model ENDPOINT_ID --deployed-model-id=ID --region=$REGION` |
| Listar custom jobs | `gcloud ai custom-jobs list --region=$REGION` |
| Acompanhar os logs de um job | `gcloud ai custom-jobs stream-logs JOB_ID --region=$REGION` |
| Listar jobs de tuning | `gcloud ai hp-tuning-jobs list --region=$REGION` |
| Construir e publicar uma imagem | `gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/ml-images/NOME:TAG .` |
| Criar repositório de pipelines KFP | `gcloud artifacts repositories create pipelines --repository-format=kfp --location=$REGION` |
| Token para chamadas REST | `gcloud auth print-access-token` |
| Agir como uma service account | `gcloud ... --impersonate-service-account=SA@PROJETO.iam.gserviceaccount.com` |

Execuções de pipelines são gerenciadas principalmente pelo SDK Python (`aiplatform.PipelineJob`) e pelo console.

### B. Glossário

| Termo | Definição |
| --- | --- |
| Agent Platform | Gemini Enterprise Agent Platform, nome do antigo Vertex AI desde abril de 2026 |
| Alias de versão | Rótulo móvel (como `champion`) que aponta para uma versão de modelo no Registry |
| Artefato | Saída de uma etapa de pipeline (dataset, modelo, métricas) com metadados e localização no Cloud Storage |
| Baseline | Modelo ou regra simples usado como referência mínima de desempenho |
| Batch inference | Previsões em lote sobre um conjunto de dados, sem endpoint permanente |
| Canary | Implantação que envia uma pequena parte do tráfego para a nova versão |
| Champion / challenger | Modelo em produção e candidato a substituí-lo |
| CMEK | Chaves de criptografia gerenciadas pelo cliente no Cloud KMS |
| Concept drift | Mudança na relação entre as entradas e o resultado que se quer prever |
| Custom job | Job de treino gerenciado que executa o seu contêiner ou pacote Python |
| Data drift | Mudança na distribuição das entradas ao longo do tempo |
| Data leakage | Uso, no treino, de informação que não estaria disponível no momento da previsão |
| Endpoint | Endereço HTTPS estável onde versões de modelos são implantadas para previsões online |
| Feature Store | Serviço que gerencia e serve features de forma consistente entre treino e produção |
| KFP | Kubeflow Pipelines, o SDK usado para definir pipelines |
| Linhagem | Registro de quais dados, código e execuções produziram um artefato |
| LLMOps | Práticas de operação para aplicações com modelos de linguagem |
| ML Metadata | Repositório de metadados de execuções, artefatos e linhagem |
| Model card | Documento que descreve o objetivo, os dados, o desempenho e as limitações de um modelo |
| Point-in-time join | Junção que traz os valores de features como estavam na data de cada exemplo |
| RAG | Geração aumentada por recuperação: o modelo responde com base em trechos recuperados dos seus dados |
| Training-serving skew | Diferença entre os dados vistos no treino e os recebidos em produção |
| VPC Service Controls | Perímetro de segurança que impede a exfiltração de dados dos serviços protegidos |
| Workload Identity Federation | Autenticação de sistemas externos (como o GitHub Actions) sem chaves de service account |

### Fontes

- [Introducing Gemini Enterprise Agent Platform](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform) — anúncio do novo nome, 22 de abril de 2026
- [Gemini Enterprise Agent Platform — notas de versão](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) — tabela de nomes antigos e novos
- [Vertex AI — notas de versão principais](https://docs.cloud.google.com/vertex-ai/docs/core-release-notes) — descontinuação da Feature Store legada e do serving Optimized
- [Introduction to Model Monitoring](https://docs.cloud.google.com/vertex-ai/docs/model-monitoring/overview) — Model Monitoring v1 e v2
- [Model versions and lifecycle](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions) — versões e aposentadoria dos modelos Gemini
- [Professional Machine Learning Engineer](https://cloud.google.com/learn/certification/machine-learning-engineer) e [guia do exame](https://services.google.com/fh/files/misc/professional_machine_learning_engineer_exam_guide_english_new.pdf)

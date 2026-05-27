# 🔄 Power Automate — Auditoria BPG: Criação Automática de Non Compliance

> Automação desenvolvida para eliminar um processo manual recorrente de auditoria mensal, comparando entregas registradas no sistema de gestão de projetos (Azure DevOps) com as entregas computadas no painel de performance (Power BI/BPG) e criando automaticamente work items de **Non Compliance** para os gestores responsáveis pelos itens divergentes.

> 📖 [Read in English](README.en.md)

---

## 🖼️ Visão Geral do Fluxo

![Flow Diagram](assets/flow-diagram.png)

---

## 💡 Contexto e Motivação

Em ciclos mensais de auditoria, era necessário identificar manualmente quais User Stories entregues no Azure DevOps não estavam refletidas no BPG (Business Performance Guide — painel Power BI), e vice-versa. Esse processo consumia tempo considerável e estava sujeito a erros humanos.

Esta automação resolve o problema de ponta a ponta: identifica as divergências, cria os cards de auditoria, gera as NCs por gestor responsável e atualiza os cards existentes caso já existam — **sem qualquer interação manual necessária**.

---

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│              TRIGGER: Recorrência Mensal                 │
└────────────────────────┬────────────────────────────────┘
                         │
            ┌────────────▼────────────┐
            │  Cálculo de Mês Dinâmico │
            │  (mês, trimestre, ano)   │
            └────────────┬────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
   ┌──────▼──────┐              ┌───────▼──────┐
   │  Escopo BPG  │              │  Escopo VSTS │
   │ (Power BI)   │              │ (Azure DevOps)│
   │ Query DAX    │              │ WIQL Query   │
   └──────┬───────┘              └──────┬───────┘
          │                             │
          └──────────┬──────────────────┘
                     │
        ┌────────────▼────────────┐
        │  Comparação de IDs      │
        │  BPG ≠ VSTS?            │
        └──────┬──────────────────┘
               │ Sim (há divergências)
    ┌──────────▼──────────┐
    │   Scope: Card Audit  │
    │ Busca ou cria card   │
    │ de auditoria do mês  │
    └──────────┬───────────┘
               │
    ┌──────────▼──────────────┐
    │ Busca lista de líderes   │
    │ (SharePoint → JSON)      │
    └──────────┬───────────────┘
               │
    ┌──────────▼───────────────────┐
    │  Busca detalhes dos cards    │
    │  divergentes + gestor de     │
    │  cada responsável            │
    └──────────┬────────────────────┘
               │
    ┌──────────▼───────────────────┐
    │  Por gestor único:            │
    │  • NC já existe? → Atualiza  │
    │  • NC não existe? → Cria     │
    └──────────────────────────────┘
```

---

## 🔌 Conectores Utilizados

| Conector | Uso |
|---|---|
| **Power BI** | Executa query DAX no dataset de performance para obter entregas do BPG |
| **Azure DevOps (VSTS)** | Consulta User Stories entregues via WIQL, busca detalhes dos cards, cria e atualiza work items |
| **SharePoint Online** | Busca lista de centros de resultado com e-mails de responsáveis e gestores |

---

## 📋 Pré-requisitos

Antes de importar e configurar este fluxo no seu ambiente, você precisa:

**Conexões Power Automate:**
- Conexão ativa com **Power BI** autenticada com um usuário que tenha acesso ao workspace e dataset
- Conexão ativa com **Azure DevOps** com permissões de leitura e escrita nos projetos envolvidos
- Conexão ativa com **SharePoint Online** com acesso à lista de centros de resultado

**Estrutura no Azure DevOps:**
- Um projeto de origem com User Stories (ex: time de engenharia)
- Um projeto de gestão com os work item types `Audit` e `Non compliance`
- Campos customizados configurados (flag de entrega de valor, campo de projeto de integração)

**Estrutura no SharePoint:**
- Uma lista com JSON contendo a matriz de responsáveis: `email`, `managerEmail`, `resultCenterName`, `businessUnitName`

**Estrutura no Power BI:**
- Dataset com tabela `fact_workitem` contendo os campos: `code_source`, `code_target`, `status`, `closed_date`, `created_date`, `target_date`, `title`, `user_account`
- Tabela de dimensão `dim_result_center_process` com `ecosystem`, `business_unit_name`, `set_process`
- Tabela de datas local com campos `Ano`, `Trimestre`, `Mês`

---

## 🔧 Guia de Configuração

### Passo 1 — Substituir os placeholders

Abra o arquivo `flow_sanitized.json` e substitua todos os valores `YOUR_*` conforme a tabela abaixo:

| Placeholder | Descrição | Exemplo |
|---|---|---|
| `YOUR_DEVOPS_ORG` | Nome da organização no Azure DevOps | `minhaempresa` |
| `YOUR_SOURCE_PROJECT` | Projeto de origem das User Stories | `MeuProduto` |
| `YOUR_MANAGEMENT_PROJECT` | Projeto onde ficam Audit e Non compliance | `GestaoPerformance` |
| `YOUR_POWERBI_WORKSPACE_GROUP_ID` | GUID do workspace no Power BI | `xxxxxxxx-xxxx-...` |
| `YOUR_POWERBI_DATASET_ID` | GUID do dataset no Power BI | `xxxxxxxx-xxxx-...` |
| `YOUR_TENANT.sharepoint.com` | URL do tenant SharePoint | `minhaempresa.sharepoint.com` |
| `YOUR_SHAREPOINT_SITE` | Nome do site SharePoint | `Performance` |
| `YOUR_RESULT_CENTER_LIST` | Nome da lista SharePoint | `ResultCenters` |
| `YOUR_AREA_PATH` | Caminho de área no Azure DevOps | `Diretoria\Engenharia` |
| `YOUR_PROJECT_NAME` | Nome do projeto para os títulos dos cards | `MeuProduto` |
| `YOUR_DELIVERY_FLAG` | Campo customizado que indica entrega de valor | `Custom.DELIVERY_FLAG` |
| `YOUR_PROJECT_FIELD` | Campo customizado com classificação de projeto | `Custom.PROJECT_TYPE` |
| `YOUR_PROJECT_1..N` | Valores do campo de projeto a incluir na query | `'PROJ_EVOLUCAO'` |
| `YOUR_EXCLUDED_AREA_1..N` | Area paths a excluir da query WIQL | `'MeuProduto\\Analise'` |
| `YOUR_RESULT_CENTER_1..N` | Nomes dos centros de resultado válidos | `'RC_ENGENHARIA'` |
| `YOUR_NC_TITLE_V1` / `YOUR_NC_TITLE_V2` | Títulos dos cards de NC a buscar/criar | `'As entregas do VSTS são iguais ao BPG?'` |
| `YOUR_BU_FIELD` | Campo customizado de BU no Azure DevOps | `BU_FIELD` |
| `YOUR_BUSINESS_UNIT_VALUE` | Valor da BU para os cards de Audit | `MinhaBU` |
| `YOUR_TEAM_TYPE` | Tipo de auditoria para o campo Type | `Engenharia` |
| `YOUR_ECOSYSTEM` | Valor do ecossistema no filtro DAX | `E-COMMERCE` |
| `YOUR_THROUGHPUT_MEASURE` | Medida de throughput no modelo Power BI | `m_average_throughput` |
| `YOUR_THROUGHPUT_TABLE` | Tabela de seleção de throughput | `select_throughput` |
| `YOUR_DATE_TABLE_ID` | Nome da tabela de datas no modelo Power BI | `LocalDateTable_xxxx` |
| `YOUR_TIMEZONE` | Fuso horário do gatilho | `E. South America Standard Time` |
| `YOUR_ENVIRONMENT_ID` | ID do ambiente Power Platform | `xxxxxxxx-xxxx-...` |

### Passo 2 — Configurar as conexões

Ao importar o fluxo no Power Automate:

1. Acesse **Power Automate → Meus Fluxos → Importar**
2. Selecione o arquivo JSON
3. Para cada conexão exibida (`shared_powerbi`, `shared_visualstudioteamservices`, `shared_sharepointonline`), selecione ou crie a conexão correspondente no seu ambiente

### Passo 3 — Validar a lista do SharePoint

A lista de centros de resultado deve ter um item com a coluna `JSON` contendo um array no formato:

```json
[
  {
    "email": "responsavel@empresa.com",
    "managerEmail": "gestor@empresa.com",
    "resultCenterName": "NOME_DO_CENTRO",
    "businessUnitName": "NOME_DA_BU"
  }
]
```

### Passo 4 — Validar campos customizados no Azure DevOps

Certifique-se de que os work item types `Audit` e `Non compliance` existem no projeto de gestão e que os campos customizados referenciados estão configurados no processo (via **Configurações da Organização → Processos**).

### Passo 5 — Configurar o dataset no Power BI

A query DAX no escopo `BPG` assume que o modelo semântico contém as tabelas e medidas listadas nos pré-requisitos. Ajuste os nomes de tabelas e medidas conforme o seu modelo antes de ativar o fluxo.

### Passo 6 — Ativar e testar

1. Salve e ative o fluxo
2. Execute manualmente uma vez clicando em **Executar** para validar o comportamento
3. Acompanhe a execução no painel de histórico do Power Automate

---

## 🔁 Lógica de Execução Detalhada

### 1. Gatilho e Inicialização de Variáveis

O fluxo é acionado mensalmente. Na inicialização, são criadas as variáveis:

- `varTotalDeItens` — inteiro para contagem geral
- `varItensFaltandoBPG` — array com IDs presentes no VSTS mas ausentes no BPG
- `varItensFaltandoVSTS` — array com IDs presentes no BPG mas ausentes no VSTS
- `varItensDetalhados` — array de objetos com detalhes de cada card divergente
- `varManagersUnicos` — array de e-mails únicos dos gestores responsáveis
- `varAuditCardId` — ID do card de auditoria do mês (criado ou encontrado)

### 2. Cálculo Dinâmico de Mês

O escopo `Escopo_-_calculo_de_mes_dinamico` calcula dinamicamente o mês anterior em três formatos:
- Nome por extenso em português (ex: `janeiro`, `fevereiro`...)
- Trimestre (ex: `Trim 1`, `Trim 2`...)
- Ano como inteiro

Esses valores são usados como filtros nas queries do Power BI e nas buscas do Azure DevOps.

### 3. Consulta ao BPG (Power BI)

O escopo `BPG` executa uma query DAX no dataset configurado filtrando pelo ecossistema, unidade de negócio e mês/trimestre/ano calculado anteriormente. O resultado é tratado para extrair apenas o ID numérico de cada item (separado por `:` no campo `code_source`), filtrando registros de subtotal (`IsGrandTotalRowTotal = false`).

### 4. Consulta ao VSTS (Azure DevOps)

O escopo `VSTS` executa uma query WIQL no projeto de origem buscando User Stories no estado diferente de `Removed`, com a flag de entrega de valor ativa, fechadas no mês anterior e pertencentes aos projetos de integração configurados, excluindo áreas específicas.

### 5. Comparação e Identificação de Divergências

O fluxo primeiro verifica se os totais já são iguais — se sim, encerra com sucesso sem processamento adicional. Caso contrário, itera sobre ambos os conjuntos de IDs e popula os arrays de itens faltando em cada sistema.

### 6. Card de Auditoria

Busca se já existe um card do tipo `Audit` para o mês atual. Se existir, reutiliza o ID; se não existir, cria um novo com o título no formato `Auditoria BPG - [PROJETO] - [MÊS_ANO]`.

Se após a comparação não houver divergências, o fluxo encerra. Caso contrário, continua para criação das NCs.

### 7. Enriquecimento dos Dados

Para cada ID divergente, busca os detalhes completos do work item no Azure DevOps e cruzar com a lista do SharePoint para identificar o e-mail do gestor responsável pelo colaborador atribuído ao card.

### 8. Criação/Atualização das NCs por Gestor

Para cada gestor único identificado:
- Busca se já existe uma NC aberta vinculada ao card de auditoria do mês
- **Se não existe:** cria um novo work item do tipo `Non compliance` com tabela HTML dos cards divergentes, vinculado hierarquicamente ao card de auditoria e atribuído ao gestor
- **Se já existe:** atualiza a descrição do card existente com a lista atualizada de divergências

---

## 📁 Estrutura do Repositório

```
.
├── flow_sanitized.json   # Definição do fluxo com dados sensíveis removidos
└── README.md             # Esta documentação
```

---

## 🧠 Técnicas e Conceitos Demonstrados

- **Automação orientada a dados** — sem intervenção manual em todo o ciclo mensal
- **Comparação entre sistemas heterogêneos** — cruzamento de dados entre Power BI (DAX) e Azure DevOps (WIQL)
- **Idempotência** — o fluxo verifica existência antes de criar, evitando duplicações
- **Agrupamento dinâmico** — NCs geradas por gestor único, consolidando múltiplos cards por responsável
- **Cálculo temporal dinâmico** — período sempre calculado relativamente ao momento de execução
- **Enriquecimento via lookup** — dados de gestores obtidos via SharePoint e cruzados em tempo de execução
- **Geração de HTML dinâmico** — tabelas estilizadas geradas programaticamente para os work items

---

## ⚠️ Observações

- O fluxo usa concorrência serial (`repetitions: 1`) nos loops principais para garantir consistência nas variáveis compartilhadas
- A query DAX usa `TOPN(502, ...)` como limite superior — ajuste conforme o volume esperado de entregas mensais
- O campo `Vsts_AssignedToEmail` é um campo específico do conector Azure DevOps para Power Automate que retorna o e-mail do responsável; verifique o nome exato no seu ambiente

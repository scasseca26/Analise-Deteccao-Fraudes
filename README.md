# Analise-Deteccao-Fraudes
Projecto de análise de fraudes em transações bancárias, focado na identificação de padrões suspeitos através de limpeza, transformação e modelagem de dados. Inclui análise exploratória e visualizações no Power BI para apoio à tomada de decisão.

![Power BI](https://img.shields.io/badge/Ferramenta-Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Status](https://img.shields.io/badge/Status-Concluído-brightgreen)

---

## Resumo Executivo

Este projecto analisa **2.512 transacções bancárias**, das quais **124 (4,94%)** foram classificadas como suspeitas pelo motor de Score de Risco desenvolvido em DAX. Em um volume total de **$747.555,57** transaccionado, identificou-se **$81.587,21 em valor de risco**. A análise revela uma concentração crítica de ataques no eixo Sul e Costa Oeste dos Estados Unidos, com o canal **ATM** como o mais vulnerável e os **Médicos** como o perfil de cliente mais visado. O horário de maior incidência é as **16h**, e três transacções foram sinalizadas para bloqueio imediato por apresentarem o Score de Risco máximo (85). O projecto foi desenvolvido inteiramente no **Power BI**, com modelagem em **Esquema Star**, tratamento de dados via **Power Query** e um motor de detecção de fraude construído com **medidas DAX**.

---

## Problema do Negócio

Uma instituição bancária necessitava de identificar e monitorizar transacções suspeitas na sua base de dados, com o objectivo de quantificar o impacto financeiro do risco, localizar os focos geográficos de ataque, identificar os canais e perfis mais vulneráveis e determinar quais transacções devem ser bloqueadas imediatamente.

O Gestor levantou as seguintes questões organizadas em três visões analíticas:

**Visão Executiva**
1. Qual é o impacto financeiro do risco?
2. Onde estão os focos geográficos de ataque?

**Visão Diagnóstico**

3. Qual é o canal mais vulnerável?
4. Quais são os perfis de clientes mais visados?
5. Em que horários a fraude acontece mais?

**Visão Intervenção**

6. Quais transacções devem ser bloqueadas imediatamente?

---

## Contexto

A base de dados utilizada contém registos de transacções bancárias com informações sobre o canal utilizado, localização, dispositivo, perfil do cliente e comportamento transaccional. Como o dataset original vinha numa única tabela, foi necessário decompô-lo em **Esquema Star** no Power Query, criando tabelas de dimensão e uma tabela facto. Foi também desenvolvido um **motor de Score de Risco** em DAX, que avalia cada transacção com base em quatro critérios comportamentais e classifica automaticamente as transacções como Seguras, Suspeitas ou Críticas.

> **Fonte dos dados:** [Kaggle](https://www.kaggle.com) — *(substitui pelo link directo do dataset)*

---

## Premissas da Análise

- Os dados foram tratados e modelados no **Power BI**.
- O dataset original (`bank_transactions_data_2.csv`) foi importado como tabela base e teve a opção **"Habilitar Carga" desactivada**, servindo apenas como fonte bruta para alimentar as tabelas do modelo, evitando duplicidade de dados.
- Uma transacção é classificada como **suspeita** quando o seu **ScoreRisk é superior a 50**.
- O **SLA de referência** para o Score de Risco foi definido com base em quatro critérios comportamentais: velocidade entre transacções, tentativas de login, duração da transacção e valor anómalo.
- A tabela `dCalendario` foi criada inteiramente em **DAX** para suportar a inteligência temporal do modelo.
- Todas as medidas DAX foram centralizadas na tabela `_Medidas` para facilitar a organização e manutenção do modelo.

---

### Transformações no Power Query

#### 1 — Importação e Tratamento da Tabela Base
- Importação do ficheiro `bank_transactions_data_2.csv`.
- Limpeza inicial de tipos de dados: `TransactionAmount` para **Decimal** e `TransactionDate` para **Data/Hora**.
- A tabela base teve a opção **"Habilitar Carga" desactivada**, servindo apenas como fonte para as restantes tabelas, sem ser carregada directamente no modelo.

#### 2 — Construção das Dimensões (Star Schema)
Para cada entidade, foi criada uma **Referência** da tabela base, isolando apenas as colunas relevantes:

- **Dim_Cliente** — Referência com as colunas de perfil do cliente.
- **Dim_Canal** — Referência com adição de uma **Coluna de Índice iniciando em 1**, nomeada `CanalID`, utilizada como Chave Substituta.
- **Dim_Localidade** — Referência com a coluna de localização.
- **Dim_Dispositivo** — Referência com as colunas de dispositivo e IP.

#### 3 — Criação de Colunas Analíticas na Fato_Transaccao
Foram criadas colunas calculadas que alimentam directamente o motor de detecção de fraude:

| Coluna | Descrição |
|--------|-----------|
| `TimeDifference` | Diferença em minutos entre `TransactionDate` e `PreviousTransactionDate`, usada para detectar velocidade anómala entre transacções |
| `Hora` | Extracção da hora a partir de `TransactionDate`, para análise de picos temporais de fraude |
| `Reference_date` | Duplicação de `TransactionDate` apenas com a data, criando uma "ponte" limpa para o relacionamento com o `dCalendario` |

---

### Medidas DAX

**Volume Total**
```dax
Volume Total = SUM(Fato_Transaccao[TransactionAmount])
```

**Qtd Transaction**
```dax
Qtd Transaction = COUNTROWS(Fato_Transaccao)
```

**Qtd Suspect**
```dax
Qtd Suspect = 
CALCULATE(
    [Qtd Transaction], 
    FILTER(Fato_Transaccao, [ScoreRisk] > 50)
)
```

**% Suspect**
```dax
% Suspect = DIVIDE([Qtd Suspect], [Qtd Transaction], 0)
```

**Value Risk**
```dax
ValueRisk = 
CALCULATE(
    [Volume Total], 
    FILTER(Fato_Transaccao, [ScoreRisk] > 50)
)
```

**% Usage Balance**
```dax
% UsageBalance = 
DIVIDE(
    SUM(Fato_Transaccao[TransactionAmount]), 
    SUM(Fato_Transaccao[AccountBalance]), 
    0
)
```

**AVG Customer Spent**
```dax
AVGCustomerSpent = CALCULATE(
    AVERAGE(Fato_Transaccao[TransactionAmount]), 
    ALLEXCEPT(Fato_Transaccao, Fato_Transaccao[AccountID])
)
```

**Alert Amount Suspect**
```dax
AlertAmountSuspect = 
VAR GastoAtual = SUM(Fato_Transaccao[TransactionAmount])
VAR MediaHist = [AVGCustomerSpent]
RETURN
IF(
    ISBLANK(GastoAtual), 
    BLANK(),
    IF(GastoAtual > MediaHist * 3, 1, BLANK())
)
```

**Change Location**
```dax
ChangeLocation = 
VAR CidadesDiferentes = DISTINCTCOUNT(Fato_Transaccao[Location])
RETURN
IF(CidadesDiferentes > 1, 1, 0)
```

**Score de Risco** *(Motor de Detecção de Fraude)*
```dax
ScoreRisk = 
IF( NOT HASONEVALUE(Fato_Transaccao[TransactionID]), BLANK(),

    VAR pVelocidade = 
        IF(MAX(Fato_Transaccao[TimeDifference]) < 60, 30, 
        IF(MAX(Fato_Transaccao[TimeDifference]) < 180, 20, 0))

    VAR pLogin = 
        IF(MAX(Fato_Transaccao[LoginAttempts]) > 3, 25, 
        IF(MAX(Fato_Transaccao[LoginAttempts]) > 2, 20, 0))

    VAR pDuracao = IF(MAX(Fato_Transaccao[TransactionDuration]) < 5, 25, 0)

    VAR pValor = IF([AlertAmountSuspect] = 1, 30, 0)

    VAR vSoma = pVelocidade + pLogin + pDuracao + pValor

    RETURN IF(vSoma = 0, BLANK(), vSoma)
)
```

**Status Risk**
```dax
Status Risk = 
VAR Score = [ScoreRisk]
RETURN
IF(
    ISBLANK(Score), 
    BLANK(),
    SWITCH( TRUE(),
        Score >= 80, "🚨 CRÍTICO",
        Score >= 50, "⚠️ SUSPEITO",
        "✅ SEGURO"
    )
)
```

**Qtd Fraud MOM** *(Variação Mês a Mês)*
```dax
Qtd Fraud MOM = 
CALCULATE(
    [Qtd Suspect], 
    DATEADD('dCalendario'[Date], -1, MONTH)
)
```

#### Lógica do Motor de Score de Risco

O `ScoreRisk` avalia cada transacção com base em **4 critérios comportamentais independentes**, cada um com uma pontuação de risco associada:

| Critério | Condição | Pontuação |
|----------|----------|-----------|
| **Velocidade** | Tempo desde última transacção < 60 min | +30 pontos |
| **Velocidade** | Tempo desde última transacção < 180 min | +20 pontos |
| **Login** | Tentativas de login > 3 | +25 pontos |
| **Login** | Tentativas de login > 2 | +20 pontos |
| **Duração** | Duração da transacção < 5 segundos | +25 pontos |
| **Valor** | Gasto > 3x a média histórica do cliente | +30 pontos |

A classificação final é atribuída pelo `Status Risk`:

| Score | Classificação |
|-------|--------------|
| ≥ 80 | 🚨 CRÍTICO |
| ≥ 50 | ⚠️ SUSPEITO |
| < 50 | ✅ SEGURO |

---

### Modelagem de Dados — Esquema Star

<img width="1237" height="720" alt="Modelagem Schema" src="https://github.com/user-attachments/assets/e4cdd701-e509-4f28-99ff-2435d67ac38c" />

A modelagem foi realizada seguindo o **Esquema Star (Star Schema)**, decompondo o dataset original numa tabela facto central rodeada por tabelas de dimensão, garantindo melhor desempenho nas consultas e maior clareza na estrutura do modelo.

### Passo 3 — Definição da Tabela Facto

A tabela facto central é a **Fato_Transaccao**, que regista cada transacção bancária como uma linha individual e serve de base para todas as métricas do motor de detecção de fraude.

| Coluna | Papel na Análise |
|--------|-----------------|
| `TransactionID` | Identificador único da transacção |
| `AccountID` | Chave estrangeira → `Dim_Cliente` |
| `TransactionAmount` | Valor da transacção — métrica financeira central |
| `TransactionDate` | Data e hora da transacção |
| `Location` | Chave estrangeira → `Dim_Localidade` |
| `DeviceID` | Chave estrangeira → `Dim_Dispositivo` |
| `TransactionDuration` | Duração da transacção em segundos — critério do Score de Risco |
| `LoginAttempts` | Número de tentativas de login — critério do Score de Risco |
| `AccountBalance` | Saldo da conta — base para o cálculo de `% UsageBalance` |
| `PreviousTransactionDate` | Data da transacção anterior — base para o cálculo de `TimeDifference` |
| `TimeDifference` | Velocidade entre transacções — critério do Score de Risco |
| `Hora` | Hora da transacção — análise de picos temporais |
| `CanalID` | Chave estrangeira → `Dim_Canal` |
| `Reference_date` | Ligação com o `dCalendario` |

### Passo 4 — Identificação das Dimensões

| Tabela de Dimensão | Colunas | Descrição |
|-------------------|---------|-----------|
| `Dim_Cliente` | `AccountID`, `CustomerAge`, `CustomerOccupation`, `AgeRange` | Perfil demográfico e profissional do cliente |
| `Dim_Canal` | `CanalID`, `Channel`, `TransactionType` | Canal e tipo de transacção |
| `Dim_Localidade` | `Location` | Localização geográfica da transacção |
| `Dim_Dispositivo` | `DeviceID`, `IP Address` | Dispositivo e endereço IP utilizados |
| `dCalendario` | `Date`, `Ano`, `MesNome`, `MesNum`, `MesAno`, `DiaSemana`, `DiaSemanaNum`, `FimDeSemana` | Inteligência temporal do modelo |

---

## Estratégia da Solução

### Passo 1 — Resumo do Contexto em Pergunta Aberta
> *Existem transacções fraudulentas na base de dados e qual é o seu impacto financeiro e operacional para a instituição?*

### Passo 2 — Transformação em Perguntas Fechadas
> - Qual é o valor total em risco nas transacções suspeitas?
> - Quais localizações geográficas concentram mais ataques?
> - Qual canal de transacção é o mais vulnerável à fraude?
> - Qual o perfil de cliente mais visado pelos atacantes?
> - Em que horários ocorre a maior concentração de transacções suspeitas?
> - Quais transacções específicas devem ser bloqueadas com urgência?

### Passo 5 — Hipóteses Analíticas

- H1: Uma parte significativa do volume transaccionado está em risco devido a transacções suspeitas.
- H2: As fraudes concentram-se geograficamente em determinadas cidades ou regiões.
- H3: O canal ATM é o mais vulnerável por ser menos supervisionado em tempo real.
- H4: Determinados perfis profissionais são mais visados devido ao seu poder de compra.
- H5: As fraudes ocorrem predominantemente em horários específicos do dia.
- H6: Transacções com Score de Risco máximo devem ser priorizadas para bloqueio imediato.

### Passo 6 — Critérios de Priorização

As hipóteses foram priorizadas com base em dois critérios:

- **Impacto Financeiro** — quanto a hipótese afecta directamente o valor em risco para a instituição.
- **Urgência Operacional** — quanto o insight exige uma acção imediata por parte da equipa de segurança.

### Passo 7 — Priorização das Hipóteses Analíticas

| Prioridade | Hipótese | Justificativa |
|------------|----------|---------------|
| Alta | H6 — Transacções a bloquear imediatamente | Acção directa e urgente para reduzir perdas |
| Alta | H1 — Impacto financeiro do risco | Quantifica a exposição total da instituição |
| Alta | H3 — Canal mais vulnerável | Orienta o reforço de segurança por canal |
| Média | H5 — Horário de maior incidência | Permite reforço de monitorização em tempo real |
| Média | H2 — Focos geográficos de ataque | Orienta acções de prevenção regionalizadas |
| Baixa | H4 — Perfil de cliente mais visado | Permite segmentação de alertas por perfil |

---

## Insights da Análise

###  Visão Executiva

**Impacto Financeiro do Risco**
Em um volume total de **$747.555,57** transaccionado, foram identificados **$81.587,21 em valor de risco**, provenientes das 124 transacções classificadas como suspeitas pelo motor de Score de Risco — representando **4,94%** do total de transacções.

**Focos Geográficos de Ataque**
Identificou-se uma **concentração crítica no eixo Sul e Costa Oeste**, com **Miami (7 ocorrências)** e **San Diego (6 ocorrências)** a liderar o ranking de localidades afectadas, sinalizando a necessidade de reforço de monitorização nestas regiões.

### Visão Diagnóstico

**Canal Mais Vulnerável**
O canal **ATM** é o mais afectado, com **42 transacções suspeitas** e **$30.988 de valor em risco**, devendo ser o foco prioritário de reforço de segurança e monitorização em tempo real.

**Perfil de Cliente Mais Visado**
Os clientes com a ocupação **Médico** são os mais visados, com **37 transacções suspeitas**, possivelmente por representarem um perfil de alto poder de compra e transacções de valor elevado.

**Horário de Maior Incidência**
As fraudes concentram-se às **16h**, com **62 transacções suspeitas** registadas nesse horário, sugerindo a necessidade de reforço de monitorização durante o período da tarde.

### Visão Intervenção

**Transacções a Bloquear Imediatamente**
As seguintes transacções apresentam o **Score de Risco máximo (85)** e devem ser bloqueadas com urgência:

| Transacção | Score de Risco | Classificação |
|------------|---------------|---------------|
| `TX000275` | 85 | 🚨 CRÍTICO |
| `TX000899` | 85 | 🚨 CRÍTICO |
| `TX001214` | 85 | 🚨 CRÍTICO |

---

##  Resultados

<img width="1329" height="743" alt="DashboardFraude1" src="https://github.com/user-attachments/assets/e9060610-82a3-45ac-b712-17de23b35cfb" />

<img width="1333" height="739" alt="DashboardFraude2" src="https://github.com/user-attachments/assets/b96f65eb-8fc4-4e4e-b13b-9529ebab420f" />

<img width="1331" height="741" alt="DashboardFraude3" src="https://github.com/user-attachments/assets/77281bff-8b18-4b3e-ab37-4697c0c06828" />


Com base na análise, é possível concluir que:

- A instituição tem **$81.587,21 em valor de risco activo**, exigindo acção imediata nas três transacções críticas identificadas.
- O reforço de segurança deve ser prioritário no canal **ATM** e nas cidades de **Miami e San Diego**, que concentram o maior número de ocorrências suspeitas.
- A monitorização em tempo real deve ser intensificada às **16h**, o horário de maior incidência de fraude.
- O perfil **Médico** deve receber alertas personalizados e limites de segurança adicionais, por ser o grupo profissional mais visado.
- O motor de **Score de Risco** desenvolvido em DAX permite à equipa de segurança priorizar automaticamente as transacções que exigem intervenção imediata, tornando o processo de triagem mais eficiente e escalável.

---

## Estrutura do Projecto

```
 Analise-Deteccao-Fraudes
│
├──  dados
│   └── bank_transactions_data_2.csv      # Base de dados original do Kaggle
├──  analise
│   └── Analise Deteccao Fraude.pbix      # Ficheiro Power BI com análises e dashboard
└── 📄 README.md                          # Documentação do projecto
```

---

## 🛠️ Ferramentas Utilizadas

- **Power BI** — Modelagem em Esquema Star via Power Query, criação de motor de detecção de fraude em DAX, visualizações interactivas e Dashboard

---

## 👤 Autor

**Santiago Casseca**
[LinkedIn](www.linkedin.com/in/santiago-casseca)

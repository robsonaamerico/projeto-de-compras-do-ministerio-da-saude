# Observatório BPS: Análise Estratégica de Compras Públicas em Saúde (2020–2026)

> **Mini-Projeto Avaliativo** — Módulo de Visualização de Dados e Business Intelligence  
> **Autor:** Robson de Almeida Américo  
> **Ferramentas:** Python (Pandas, Pathlib), Jupyter Notebook (`Projeto.ipynb`), Microsoft Power BI, Power Query (M), DAX  
> **Base de Dados:** Banco de Preços em Saúde (BPS) — Ministério da Saúde do Brasil  

---

## 1. Objetivo do Projeto

O objetivo principal deste projeto é desenvolver uma solução analítica completa de Business Intelligence para monitorizar, auditar e avaliar a eficiência das aquisições públicas de medicamentos e dispositivos de saúde homologadas no **Banco de Preços em Saúde (BPS)** entre os anos de **2020 e 2026**.

A solução visa:
* Proporcionar transparência ativa sobre a alocação de recursos públicos no Sistema Único de Saúde (SUS).
* Avaliar o comportamento de preços unitários em relação aos volumes adquiridos (ganhos e economia de escala).
* Identificar a concentração orçamentária por regiões, instituições demandantes, distribuidores e laboratórios fabricantes.
* Servir como ferramenta de apoio à decisão para gestores públicos hospitalares e órgãos de controlo na formulação de estimativas e termos de referência de licitação.

---

## 2. Contextualização do Problema de Negócio

A aquisição de produtos farmacêuticos e hospitalares pelo setor público no Brasil envolve milhares de processos licitatórios descentralizados em âmbito federal, estadual e municipal. Esse ambiente descentralizado apresenta desafios estruturais:
* **Assimetria de Informação:** Disparidades substanciais de preços pagos pelo mesmo princípio ativo entre diferentes entes públicos.
* **Complexidade Licitatória:** Predomínio de modalidades específicas (como Pregão Eletrônico e Sistema de Registro de Preços) com dinâmicas próprias de fixação de margem e lote.
* **Integridade dos Dados Públicos:** Bases legadas abertas frequentemente contêm erros materiais de preenchimento (ex.: inversão de separadores decimais, inserção de códigos de erro como sentinelas numéricas e digitações com escala multiplicada), exigindo rigor técnico de auditoria e saneamento de dados antes de qualquer visualização executiva.

---

## 3. Fonte de Dados e Dicionário de Metadados

Os dados brutos foram obtidos a partir do **Portal Brasileiro de Dados Abertos**:
* **Conjunto de dados:** Banco de Preços em Saúde (BPS)
* **URL de Acesso:** [https://dadosabertos.saude.gov.br/dataset/bps](https://dadosabertos.saude.gov.br/dataset/bps)
* **Recorte Analítico:** Compras homologadas no intervalo de **2020 a 2026** (arquivos anuais estruturados em formato `.csv`).

### Principais Campos Utilizados no Modelo:

| Campo Original | Tipo | Descrição no Negócio |
| :--- | :--- | :--- |
| `co_seq_bps` | Inteiro | Identificador sequencial único da transação homologada no BPS. |
| `ano_compra` | Inteiro | Ano de efetivação da compra pública (2020–2026). |
| `sg_uf` | Texto | Sigla da Unidade Federativa da entidade compradora. |
| `no_municipio` | Texto | Município da instituição compradora. |
| `no_instituicao` | Texto | Razão social do hospital, secretaria ou órgão adquirente. |
| `modalidade` | Texto | Modalidade licitatória (ex.: Pregão, Registro de Preços, Dispensa). |
| `ds_item` | Texto | Descrição padronizada do medicamento ou dispositivo médico. |
| `no_fornecedor` | Texto | Razão social da empresa distribuidora ou fornecedora vencedora. |
| `no_fabricante` | Texto | Indústria ou laboratório farmacêutico fabricante do item. |
| `qt_medicamento` | Decimal | Volume físico total de unidades adquiridas no lote. |
| `vl_preco_unitario` | Moeda | Preço unitário homologado pago por unidade do insumo. |
| `vl_preco_total` | Moeda | Montante financeiro total contratado na transação. |

---

## 4. Engenharia de Dados: Concatenação, Diagnóstico e Tratamento Inicial (Python)

A primeira etapa da engenharia de dados foi executada em Python através do notebook `Projeto.ipynb`, garantindo rastreabilidade, velocidade no processamento de grandes volumes e validação de consistência antes da ingestão no Power BI.

### 4.1. Unificação Estrutural dos Arquivos CSV
Os microdados anuais brutos do BPS apresentavam separação estruturada por ponto e vírgula (`;`). Foi desenvolvido um script utilizando a biblioteca `pandas` e `pathlib` para concatenar os arquivos anuais (2020 a 2026) em um arquivo consolidado (`todos.csv`), prevenindo leituras recursivas do arquivo de saída:

```python
import pandas as pd
from pathlib import Path

pasta = Path(".")
arquivos = sorted(
    arquivo for arquivo in pasta.glob("*.csv")
    if arquivo.name not in ["todos.csv", "todos_tratado.csv"]
)

# Concatenação preservando o delimitador de ponto e vírgula (;)
dados = pd.concat(
    [pd.read_csv(arquivo, sep=";", low_memory=False) for arquivo in arquivos],
    ignore_index=True
)

dados.to_csv("todos.csv", index=False, sep=";")
```

### 4.2. Diagnóstico de Qualidade e Tratamento de Nulos/Duplicatas
A leitura e auditoria da base unificada `todos.csv` revelou o seguinte cenário de qualidade:
* **Dimensão:** 367.570 linhas e 36 colunas.
* **Duplicatas Completas:** **0 linhas** integralmente duplicadas.
* **Valores Nulos:** Concentrados em colunas administrativas específicas (`nu_ata`, `vl_capacidade`, `sg_unidade_medida`, `registro_anvisa`, `fg_generico`, `ds_observacao`, `no_instituicao`).

**Decisão Metodológica de Tratamento:**  
Optou-se pela **não exclusão de linhas com nulos**, uma vez que a eliminação resultaria em descarte desnecessário de transações legítimas. A imputação de valores seguiu o significado semântico de cada atributo:

| Coluna | Estratégia Adotada | Justificativa Técnica / Regra de Negócio |
| :--- | :--- | :--- |
| `nu_ata`, `nu_processo_compra` | `"Não informado"` | Campos ausentes quando a contratação não deriva de ata de SRP. |
| `sg_unidade_medida` | `"Não se aplica"` | Registros com unidade de dispensação não codificada no catálogo. |
| `registro_anvisa`, `fg_generico` | `"Não informado"` | Preserva a integridade da compra mesmo sem metadado sanitário explícito. |
| `ds_observacao` | `"Sem observação"` | Campo opcional de preenchimento na transmissão do processo licitatório. |
| `no_instituicao` | `"Não informado"` | Transações com omissão de razão social da unidade compradora. |
| Colunas de classificação (`co_grupo`, `no_grupo`) | `"Não classificado"` | Itens sem classificação hierárquica no catálogo mestre. |
| `vl_capacidade` | **Manter Nulo (`null`)** | A aplicação de média/mediana criaria dados artificiais falsos para itens sem capacidade definida. |

```python
# Script de imputação e geração do arquivo sanitizado todos_tratado.csv
colunas_texto = [
    "nu_ata", "sg_unidade_medida", "registro_anvisa", 
    "fg_generico", "ds_observacao", "no_instituicao", "nu_processo_compra"
]

for col in colunas_texto:
    dados[col] = dados[col].fillna("Não informado")

dados["ds_observacao"] = dados["ds_observacao"].replace("Não informado", "Sem observação")
dados["sg_unidade_medida"] = dados["sg_unidade_medida"].fillna("Não se aplica")

dados.to_csv("todos_tratado.csv", index=False, sep=";")
```

---

## 5. Pipeline ETL no Power BI e Saneamento Crítico de Anomalias (Outliers)

Após a importação de `todos_tratado.csv` para o Power BI, foi identificado um comportamento anômalo crítico nas métricas consolidadas: o montante do KPI de **Valor Total** acumulava cifras irreais na casa dos **trilhões de reais** (`R$ 6 Tri` a `R$ 10 Tri`), colapsando as coordenadas dos visuais analíticos.

### 5.1. Investigação da Causa Raiz
1. **Divergência de Formatação Regional (Localidade):** Campos decimais extraídos no padrão internacional com ponto decimal (`en-US`) sofriam conversão inadequada na etapa `Tipo Alterado com Localidade` quando interpretados no padrão `pt-BR`, fazendo com que os centavos fossem computados como centenas de milhares.
2. **Valores Sentinela e Inconsistências na Origem:** Identificaram-se registros com digitações extremas transmitidas por hospitais ao BPS, tais como compras isoladas com preços unitários de `R$ 2.944.000,00` e faturas totais registrando mais de `R$ 73.000.000.000,00` por erro grosseiro de digitação de lote.

### 5.2. Ações de Saneamento no Power Query
Para restabelecer a precisão matemática sem distorcer o histórico do SUS:
* **Padronização Regional:** Configuração explícita da tipagem das colunas numéricas com a localidade correta (`Inglês - Estados Unidos`).
* **Filtros de Sanidade Operacional:**
  * Delimitação de `vl_preco_unitario`: maior que R$ 0,00 e menor ou igual a **R$ 100.000,00** (cobertura suficiente para qualquer terapia biológica ou medicamento de alta complexidade legítimo do SUS).
  * Delimitação de `vl_preco_total`: compras de lote contidas entre R$ 0,01 e **R$ 50.000.000,00**.
* **Impacto do Saneamento:** Os KPIs estabilizaram-se nos montantes orçamentários reais do Ministério da Saúde:
  * **Valor Total:** Estabilizado em **R$ 236 Bi** (soma legítima do período 2020–2026).
  * **Volume Transacionado:** **29,5 Bi** de itens.
  * **Registros de Compra Válidos:** **358 Mil** compras (preservação de mais de 97% da base bruta).
  * **Preço Médio Ponderado:** **R$ 8,009** por unidade.

---

## 6. Modelagem de Dados e Medidas DAX

A arquitetura semântica foi construída com foco em performance e consistência relacional, concentrando os cálculos analíticos em uma tabela dedicada (`_Medidas`).

### Principais Medidas DAX Desenvolvidas:

* **Valor Total Acumulado:**
  ```dax
  Valor Total = SUM(todos_tratado[vl_preco_total])
  ```
* **Quantidade Total de Itens:**
  ```dax
  Quantidade Total = SUM(todos_tratado[qt_medicamento])
  ```
* **Total de Registros de Compra:**
  ```dax
  Total Registros = COUNTROWS(todos_tratado)
  ```
* **Instituições Compradoras Únicas:**
  ```dax
  Total Instituicoes = DISTINCTCOUNT(todos_tratado[no_instituicao])
  ```
* **Fornecedores Únicos Atendidos:**
  ```dax
  Total Fornecedores = DISTINCTCOUNT(todos_tratado[no_fornecedor])
  ```
* **Preço Unitário Médio Ponderado:**
  Calculado pela razão entre a receita total e o volume físico, evitando distorções provocadas por médias aritméticas simples de itens com volumes heterogêneos:
  ```dax
  Preco Medio Ponderado = DIVIDE([Valor Total], [Quantidade Total], 0)
  ```

---

## 7. Arquitetura do Relatório e Análise Visual

O relatório no Power BI foi estruturado em **5 páginas temáticas**, com cabeçalho estático padronizado (contendo os 5 segmentadores sincronizados e os 6 cartões de KPI consolidados):

1. **Página 1 — Evolução Temporal:** Gráfico de eixo duplo relacionando o gasto orçamentário anual (colunas) com o Preço Médio Ponderado (linha), além da rosca com distribuição percentual por modalidade de licitação (destaque absoluto para o Pregão Eletrônico com mais de 90% do volume).
2. **Página 2 — Estados, Municípios e Instituições:** Painel geográfico hierárquico com o ranking de dispêndio financeiro por estado e municípios demandantes.
3. **Página 3 — Medicamentos e Dispositivos:** Curva ABC de itens por relevância financeira, identificando os princípios ativos que mais pressionam o orçamento público.
4. **Página 4 — Fornecedores e Fabricantes:** Análise de concentração industrial e comercial, mapeando as principais distribuidoras contratadas e os laboratórios fabricantes de maior volume.
5. **Página 5 — Variação de Preço Unitário:** Gráfico de dispersão (*scatter chart*) com **escala logarítmica** cruzando volume adquirido versus preço unitário médio, explicitando o comportamento da curva de economia de escala (compras em grandes lotes associadas a menores preços unitários).

---

## 8. Ressalva Metodológica de Preços

> **Nota Metodológica Obrigatória:**  
> A verificação de discrepâncias ou variações de preços unitários no BPS não configura, por si só, irregularidade, sobrepreço ou superfaturamento. As variações observadas decorrem de especificidades contratuais, diferenças nas formas de apresentação e dosagem farmacêutica, condições e prazos de entrega, custos logísticos de distribuição regional e vantagens econômicas obtidas em contratações de grande escala.

---

## 9. Instruções de Reprodução

1. Baixe os CSVs anuais de 2020 a 2026 no diretório raiz do projeto.
2. Execute o notebook `Projeto.ipynb` para unificar e sanear a base bruta gerando `todos_tratado.csv`.
3. Abra o arquivo `Dashboard.pbix` no **Power BI Desktop**.
4. Em **Transformar Dados > Configurações da Fonte de Dados**, atualize o caminho para apontar para o seu arquivo `todos_tratado.csv` local e clique em **Atualizar**.
   

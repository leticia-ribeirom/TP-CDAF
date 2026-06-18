# Dados e Feature Engineering

Documenta a camada de dados: da base bruta ao dataset pronto para modelagem.
Notebooks de referência: `exploratory_analysis.ipynb` e `feature_engineering.ipynb`.

## 1. Fonte e integração

- **Fonte:** dump estruturado do **Transfermarkt** (padrão `dcaribou/transfermarkt-datasets`),
  com clubes, jogadores, transferências, valuations, competições.
- A EDA (`exploratory_analysis.ipynb`) faz a carga dos dados brutos (JSONL → DataFrame unificado),
  trata clubes/jogadores/competições e exporta as bases consolidadas em `output/`:
  - `clubs_consolidated.csv`, `players_consolidated.csv`, `competitions_consolidated.csv`,
    `transfers_consolidated.csv`, `transfers_enriched_consolidated.csv`.

## 2. Funil de filtragem

Partimos de **44.627 movimentações brutas** (8 temporadas; inclui empréstimos, transferências
gratuitas, fim de empréstimo). Para estudar prêmio, mantemos apenas **compras pagas** com `fee` e
`market_value` válidos:

| Filtro | Motivo |
|--------|--------|
| Apenas `fee_type = paid` | Empréstimos e transferências gratuitas não têm preço negociado |
| `fee ≥ €250k` | Abaixo desse limiar o `premium_ratio` tem mediana ≤ −82% e desvio < 0,30 → **sem dinâmica real de mercado** (ruído administrativo) |
| `market_value` válido | É a base do "preço justo" |

Resultado (base de **8 temporadas**, 44.627 movimentações brutas): após filtros e limpeza, o
dataset de modelagem fica com **5.146 transferências** × 8 temporadas (2017–2025, sem 2020) × 7
ligas. *(Na versão de 3 temporadas eram ~2.079.)*

## 3. Tratamento de distribuição

- **Winsorização** do `premium_ratio` nos percentis **1–99**. A cauda chega a **+3900%**, o que
  distorceria qualquer estimador. Gera `premium_ratio_w`.
- **Transformações log** para corrigir a forte assimetria à direita dos valores monetários
  (mediana €4,5M vs. média €10,3M): `log_fee = ln(1+fee)`, `log_mv = ln(1+market_value)`.

## 4. Features derivadas do clube (por clube × temporada)

Capturam a hipótese central e o confundidor "clube rico":

| Feature | Significado |
|---------|-------------|
| `revenue_sales` | Receita total de vendas na janela — **variável de tratamento central** (vira `D = ln(1+revenue)` na Etapa 2) |
| `n_sales` | Número de vendas (intensidade da atividade vendedora) |
| `max_sale` | Maior venda individual (efeito de vendas *blockbuster*) |
| `total_spend` | Gasto total em compras — controla C1 (clube rico compra e vende muito) |
| `n_buys` | Número de compras |
| `net_balance` | Receita − gasto (modo "investimento" vs. "realização de lucro") |
| `net_transfer_record` | Saldo líquido de transferências |

A separação entre `revenue_sales` (caixa recente) e `total_spend` (comportamento habitual de gasto)
é o que permite distinguir o efeito "ter dinheiro em caixa" do viés de "clube rico".

## 5. Features de rede — grafo de transferências

O mercado é modelado como um **grafo dirigido e ponderado**, reconstruído **por temporada**
(`networkx`):

- **Nós** = clubes · **Arestas** = transferências (vendedor → comprador) · **Peso** = fee em euros.

Para cada clube em cada temporada extraímos:

| Métrica | Significado |
|---------|-------------|
| `in_degree` | Nº de vendedores distintos que forneceram jogadores ao clube |
| `out_degree` | Nº de compradores distintos que compraram do clube |
| `pagerank` | Importância estrutural na rede (comprar de clubes importantes eleva o PR) |
| `in_strength` | Volume total (€) gasto em compras |
| `out_strength` | Volume total (€) recebido em vendas |
| `net_flow` | `out_strength − in_strength` (positivo = exportador líquido) |

Essas métricas capturam **poder estrutural / de barganha** — um confundidor relevante (PSG,
Chelsea, Real Madrid dominam `in_strength`), e são reaproveitadas na Etapa 2 como confundidores
(W) e na análise de heterogeneidade do IVB (CATE × PageRank).

## 6. Variável dependente (4 variantes)

A feature engineering exporta quatro versões do alvo, para flexibilidade na modelagem:

| Variante | Descrição |
|----------|-----------|
| `premium_ratio` | Prêmio bruto: (fee − MV) / MV |
| `premium_ratio_w` | Versão winsorizada |
| `hedonic_residual` | Resíduo log de uma regressão hedônica OLS exploratória |
| `pi_hedonic_pct` | Resíduo convertido para percentual |

> Observação: a feature engineering inclui um modelo hedônico **OLS** exploratório para gerar um
> resíduo de referência. A versão **oficial e final** do resíduo (`premio_reinvestimento`) é a
> produzida pelo **Random Forest** na Etapa 1 — ver [hedonic_ml.md](hedonic_ml.md).

## 7. Saídas

- `output/transfers_modeling_ready.csv` — dataset final da feature engineering (~5.146 linhas).
- `output/transfers_etapa2_ready.csv` — dataset com o resíduo do RF (`premio_reinvestimento`),
  53 colunas, 5.146 linhas (entrada da Etapa 2).

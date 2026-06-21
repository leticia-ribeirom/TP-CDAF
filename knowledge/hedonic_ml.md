# Etapa 1 — Modelo Hedônico (Machine Learning)

Notebook: `notebooks/etapa1_hedonic_model.ipynb`
Entrada: `output/transfers_modeling_ready.csv` · Saída: `output/transfers_etapa2_ready.csv`

> ⚠️ **Atualizado para 8 temporadas.** Split agora é treino 2017–2024 / teste 2025, com dummies de
> temporada dinâmicas (corrigido o hardcode de 2024/2025). Números abaixo já refletem o re-run.
> Changelog: [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md).

## 1. Objetivo

Estimar o **preço justo** de cada transferência a partir dos atributos do jogador e de controles
estruturais de mercado. O que o modelo **não consegue explicar** — o **resíduo** — é o
**"prêmio de reinvestimento"**, que vira a variável de resultado (Y) da Etapa 2.

$$Y_{i,c} = \ln(P_i) - \ln(\hat{P}_i)$$

onde `P_i` é o fee pago e `P̂_i` é o preço previsto pelo modelo. Por construção, `Y` está livre
das características intrínsecas do atleta; o que sobra são as **forças conjunturais da negociação**.

## 2. Por que um modelo hedônico?

Um "modelo hedônico" decompõe o preço de um bem na soma do valor de seus atributos. Aqui, o preço
de um jogador é função de idade, posição, valor de mercado, liga e temporada. Remover essa parcela
"justa" é o que permite isolar o sobrepreço — caso contrário, confundiríamos "pagou caro porque o
jogador é bom" com "pagou caro porque estava pressionado".

## 3. Variáveis

**Alvo (target):** `log_fee = ln(1+fee)`.

**Covariáveis (19 features) — apenas jogador + contexto estrutural, nunca finanças do comprador:**

| Feature | Papel |
|---------|-------|
| `age`, `age_sq` | Ciclo biológico do jogador — relação **não-linear** (pico de valor por volta dos 20, declínio depois). O termo quadrático captura a curvatura. |
| `log_mv` | Logaritmo do valor de mercado (Transfermarkt) — o "preço justo" de referência. |
| `is_attacker`, `is_midfielder`, `is_defender` | Grupo de posição. |
| 6 dummies de liga (`competition_code_*`) | Nível/poder econômico da liga. |
| 7 dummies de temporada (`season_2018` … `season_2025`) | Efeitos fixos de temporada (absorvem inflação de mercado). Base = 2017. |

> 6 (jogador) + 6 (liga) + 7 (temporada) = **19 features**. As dummies de temporada eram
> hardcoded em 2 (2024/2025); foram corrigidas para cobrir todas as 8 temporadas.

> **Princípio de design:** nenhuma feature pode descrever o **comportamento financeiro do clube
> comprador**. Essa separação é o que garante que o resíduo capture apenas a conjuntura da
> negociação — o objeto da Etapa 2. As features de clube/rede ficam reservadas para os
> confundidores (W) da Etapa 2.

## 4. Split treino/teste — temporal

- **Treino:** temporadas **2017–2024** (4.396 obs, 85%).
- **Teste (holdout):** temporada **2025** (750 obs, 15%).

O split é **temporal** (não aleatório) de propósito: simula o uso real — treinar no passado e
prever o futuro — e é um teste de generalização mais honesto.

## 5. Modelos e resultados

Quatro modelos comparados por validação cruzada (5-fold no treino) e no holdout. Métrica principal:
**RMSE** (penaliza erros grandes — relevante quando os fees vão de €250k a €100M).

| Modelo | CV RMSE (μ) | CV R² (μ) | Test RMSE | Test MAE | Test R² |
|--------|-------------|-----------|-----------|----------|---------|
| XGBoost | 0,686 | 0,735 | 0,741 | 0,548 | 0,711 |
| LightGBM | 0,652 | 0,760 | 0,703 | 0,524 | 0,739 |
| **Random Forest** ✅ | **0,674** | **0,731** | **0,673** | **0,490** | **0,762** |
| SVR (RBF) | 0,678 | 0,741 | 0,729 | 0,540 | 0,720 |

- **Vencedor: Random Forest** (menor Test RMSE = 0,673; maior Test R² = 0,762; OOB R² = 0,735).
- O gap pequeno entre CV e teste em todos os modelos indica **ausência de overfitting relevante**.

## 6. Interpretabilidade (SHAP)

SHAP aplicado ao melhor modelo (Random Forest). Ranking de importância:

1. **`log_mv`** — dominante (o valor de mercado é, de longe, o maior preditor do fee).
2. `age` e `age_sq`.
3. dummies de liga.
4. posição, temporada.

Isso **valida empiricamente a literatura hedônica**: o valor de mercado do Transfermarkt é um
proxy forte do preço, e idade/liga refinam a estimativa.

## 7. O resíduo (saída para a Etapa 2)

O resíduo `premio_reinvestimento` é calculado e exportado em `transfers_etapa2_ready.csv`
(colunas adicionadas: `log_fee_hat`, `premio_reinvestimento`).

Distribuição do resíduo (n = 5.146):

| Estatística | Valor |
|-------------|-------|
| média | −0,001 (≈ 0, como esperado) |
| mediana | **+0,035** |
| desvio-padrão | 0,671 |
| mínimo / máximo | −4,47 / +2,64 |
| **% com prêmio positivo** | **52,5%** |

Ou seja: pouco mais da metade das transferências saem **acima** do preço justo estimado, com um
sobrepreço mediano modesto. Esse resíduo é a matéria-prima da análise causal.

> **Nota:** o resíduo é gerado **out-of-fold** (`cross_val_predict`), sem vazamento in-sample. O
> desvio-padrão (0,671) é maior que na versão anterior (0,587) justamente porque a variância deixou
> de ser artificialmente comprimida.

## 8. Pontos de atenção

- O modelo usa **só atributos básicos** do jogador. Métricas de performance (gols, assistências,
  xG, minutagem do FBref/StatsBomb), prometidas na proposta, **não foram integradas** — ver
  [pendencias.md](pendencias.md). Isso limita o teto do R² e deixa parte do "preço justo" por
  explicar (potencialmente contaminando o resíduo).
- **Tuning de hiperparâmetros (Optuna)** não foi feito; os modelos rodam com configurações padrão.

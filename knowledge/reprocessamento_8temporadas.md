# Reprocessamento com 8 temporadas — o que mudou nos números e insights

Documento-changelog do re-processamento de todo o pipeline após expandir a base de
**3 temporadas (2023–2025)** para **8 temporadas (2017–2025, sem 2020)**.

> **Resumo de uma frase:** mais dados **enfraqueceram a descoberta principal** — o "prêmio do
> vendedor" médio, que era *no limiar* da significância com 3 temporadas, virou **não significativo**
> com 8. Em compensação, surgiu uma história mais rica e honesta: **o efeito é concentrado em
> janelas de mercado aquecido (2022–2023)** e a **liquidez defasada** mostra efeito significativo
> (enfraquece causalidade reversa).

## 1. Como foi feito

- **Dados novos:** raspados com `scripts/automate_crawls_new.py --skip-players` (só `transfers` +
  `clubs`, pois players não alimenta o modelo — ver [dados_e_features.md](dados_e_features.md)).
  Temporadas adicionadas: 2017, 2018, 2019, 2021, 2022. (2020 não foi coletada → há um buraco.)
- **Pipeline re-rodado na ordem:** `exploratory` (consolidação) → `feature_engineering` →
  `etapa1` → etapa2 (script manual) → `etapa2_robustez`.

### Correções de código necessárias (bugs de "parametrizado para 3 temporadas")
O re-run **não foi mecânico** — dois hardcodes teriam invalidado os resultados:
1. **`etapa1`** usava `cols_season = ['season_2024','season_2025']` e split fixo `treino=2023-24,
   teste=2025`. Corrigido para **dummies de temporada dinâmicas** (todas as 8) e **treino = todas
   as temporadas < 2025, teste = 2025**. Sem isso, 2017–2023 ficavam sem controle de inflação e os
   resíduos saíam enviesados.
2. **`etapa2` e `etapa2_robustez`** tinham o mesmo `CONFOUNDERS` com season hardcoded; corrigido
   para incluir **todas** as dummies de temporada. No `robustez`, a regra do lag (`drop 2023`)
   também foi corrigida para manter só temporadas cujo `t-1` existe na base.

### Restrição de ambiente (importante)
O **`econml` não instala neste ambiente** (Python 3.12 + NumPy 2.4 → build Cython falha). Por isso:
- A **etapa 2 foi recalculada com DML manual** (cross-fitting + *partialling-out*, HC1), a mesma
  implementação **validada** no `etapa2_robustez` (reproduz o ATE do econml).
- O **CATE/IVB** foi estimado por **R-learner** (Nie & Wager), que **não é idêntico** ao
  `CausalForestDML` do econml. Logo, **os números de IVB não são diretamente comparáveis** aos
  da versão antiga — mudaram tanto por mais dados quanto por troca de método.

## 2. Quadro comparativo (3 temporadas → 8 temporadas)

### Base e Etapa 1 (hedônico)
| Item | 3 temporadas | 8 temporadas |
|------|--------------|--------------|
| Transferências brutas | 16.215 | 44.627 |
| Compras pagas (modeling-ready) | 2.144 | 5.146 |
| Split treino/teste | 2023–24 / 2025 | **2017–24 / 2025** |
| Melhor modelo | Random Forest | Random Forest |
| **Test R²** | 0,753 | **0,762** |
| Test RMSE | 0,685 | 0,673 |
| Dummies de temporada (FE) | 2 | **7** |
| % com prêmio positivo | 51,6% | 52,5% (out-of-fold) |
| Prêmio mediano (resíduo) | +0,018 | +0,035 (out-of-fold) |

➡️ A Etapa 1 ficou praticamente **igual em qualidade** (log_mv domina), mas agora é **legítima
sobre 8 temporadas**, com inflação de mercado controlada (7 dummies de temporada; total de **19
features**, antes 14 com o hardcode).

> **Bônus do re-scrape:** a base de `clubs` reconsolidada veio completa → a **imputação de variáveis
> de elenco caiu para 0 nulos**. A limitação antiga ("706/750 obs de 2025 imputadas") **deixou de
> existir**.

### Etapa 2 (causal) — a mudança que importa
| Item | 3 temporadas | 8 temporadas |
|------|--------------|--------------|
| **ATE (θ)** | **0,0118** | **0,0044** |
| IC 95% (HC1) | [0,0003; 0,0232] | [−0,0016; 0,0104] |
| IC 95% (cluster) | — | [−0,0021; 0,0110] |
| **Significativo a 5%?** | Sim (no limiar) | **NÃO** (HC1 e cluster) |
| Placebo shuffle | −0,0027 | 0,0024 |
| Placebo noise | −0,0027 | 0,0009 |
| R² 1ª etapa (model_t) | 0,60 | 0,37 |
| R² 1ª etapa (model_y) | 0,11 | 0,06 |
| CATE — faixa | [−0,089; +0,171] (CausalForest) | [−0,39; +2,25] (R-learner) |
| IVB — top "presa" | FC Arouca (0,81) | Nîmes Olympique (0,52) |

➡️ **O efeito causal médio encolheu para perto de zero e perdeu a significância** (com HC1 e com SE
clusterizado). O placebo shuffle (0,0024) ficou quase do tamanho do ATE (0,0044) — forte sinal de
que, *na média*, o efeito é nulo com a base ampliada. (Valores sobre o Y **out-of-fold**, sem
vazamento; o ATE subiu levemente vs. a versão in-sample anterior.)

### θ por temporada — onde o efeito realmente vive
| Temporada | θ | IC 95% | Significativo? |
|-----------|-----|--------|----------------|
| 2017 | −0,006 | [−0,026; 0,014] | não |
| 2018 | −0,013 | [−0,036; 0,011] | não |
| 2019 | +0,024 | [−0,011; 0,058] | não |
| 2021 | +0,012 | [−0,001; 0,025] | não |
| **2022** | **+0,064** | [0,028; 0,101] | **SIM** (p_bonf=0,005) |
| **2023** | **+0,047** | [0,017; 0,077] | **SIM** (p_bonf=0,019) |
| 2024 | +0,003 | [−0,053; 0,059] | não |
| 2025 | +0,015 | [−0,010; 0,040] | não |

(IC clusterizado por clube; 2022/2023 **sobrevivem a Bonferroni e FDR**.)

➡️ **O "prêmio do vendedor" não é uma lei universal — ele aparece nos anos de mercado aquecido
(2022 e 2023, o boom pós-COVID) e some no resto.** A análise antiga (2023–2025) "pegou" justamente
o 2023 forte, o que explica por que o efeito parecia significativo com 3 temporadas.

### Teste de lag (causalidade reversa, C2) — enfraquece C2, mas não conclusivo
(SE clusterizado clube×temporada)

| Especificação | n | θ | IC 95% | Significativo? |
|---------------|---|-----|--------|----------------|
| Contemporâneo (amostra completa) | 5.146 | 0,0044 | [−0,0021; 0,0110] | não |
| Contemporâneo (subamostra com lag) | 4.008 | 0,0055 | [−0,0027; 0,0136] | não |
| **DEFASADO t-1** | 4.008 | **0,0051** | **[0,0012; 0,0090]** | **SIM** |

➡️ A liquidez defasada tem efeito positivo significativo — e o ano anterior não pode ter sido
causado pela compra atual → **enfraquece a causalidade reversa (C2)**. **Ressalva honesta:** na
mesma subamostra o contemporâneo é n.s. mas o lag é significativo, o que sugere que o lag capta um
**traço persistente de clube** (corr=0,44), não um efeito limpo. **Não é conclusivo.**

## 3. Leitura — como isso muda a história

1. **A manchete muda de "existe um prêmio causal" para "o prêmio é condicional ao regime de
   mercado".** Liderar pela significância do ATE agora seria **incorreto** — o ATE não é
   significativo. A narrativa honesta é a da **heterogeneidade temporal**: o prêmio emerge quando
   o mercado está aquecido (2022–2023).
2. **A heterogeneidade entre clubes (IVB) continua existindo**, mas os números mudaram de método
   (R-learner) e de composição (clubes menores no topo). Tratar o IVB como **ilustrativo**, não
   como ranking definitivo, até reprocessar com `CausalForestDML`.
3. **O teste de lag dá suporte modesto** à direção causal (enfraquece C2), mas **não é conclusivo**
   — o contemporâneo é n.s. na mesma amostra, sinal de contaminação por traço de clube.
4. **O "mecanismo de sinalização" (D₂/D₃) é exploratório, não confirmatório:** significativo no
   bruto, mas **perde a significância** ao controlar intensidade de janela (clube ativo).

## 4. Ressalvas / pendências abertas deste reprocessamento

- **IVB precisa do econml** para ser comparável ao desenho original (`CausalForestDML`). O R-learner
  é um substituto válido, mas diferente. Rodar em ambiente com econml (ex.: Colab / Python 3.12)
  fecharia isso.
- ✅ **`etapa2_double_ml.ipynb` foi portado para DML manual** e agora **executa sem econml**
  (Robinson + cross-fitting + HC1; CATE via R-learner). Os números do notebook batem com o script.
  Resta apenas, se quiser o IVB idêntico ao desenho original, rodar o `CausalForestDML` em ambiente
  com econml.
- **2020 ausente** (buraco na série) e **2021 é regime COVID** — ambos entram nos efeitos fixos de
  temporada, mas valem nota no relatório.
- Resíduos da Etapa 1 são em parte **in-sample** (treino) — herdado do desenho original; considerar
  resíduos *cross-fitted* numa próxima iteração.

Ver também: [resumo.md](resumo.md), [hedonic_ml.md](hedonic_ml.md), [double_ml.md](double_ml.md),
[pendencias.md](pendencias.md).

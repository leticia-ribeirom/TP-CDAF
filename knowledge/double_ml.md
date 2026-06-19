# Etapa 2 — Modelagem Causal (Double Machine Learning)

Notebook: `notebooks/etapa2_double_ml.ipynb`
Entrada: `output/transfers_etapa2_ready.csv` (5.146 transferências)
Saída: `output/resultados_etapa2_ivb.csv` (ranking IVB de 190 clubes)
Spec original: `deliverables/etapa2_modelagem_causal (1).docx`

> ⚠️ **Atualizado para 8 temporadas — e a conclusão mudou.** Com mais dados o **ATE deixou de ser
> significativo** (era 0,0118\* com 3 temporadas; agora 0,0044 n.s. com HC1 e com SE clusterizado).
> O efeito é significativo só em 2022–2023 (e sobrevive a Bonferroni/FDR).
>
> 🔧 **Notebook portado:** como o `econml` não instala no ambiente (Python 3.12 + NumPy 2.x), o
> `etapa2_double_ml.ipynb` foi **portado para Double ML manual** (Robinson + cross-fitting + HC1) e
> **roda sem econml**. O CATE/IVB usa **R-learner** (aproximação do CausalForestDML). Texto abaixo
> descreve o **desenho**; números atuais e ressalvas no changelog:
> [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md).

## 1. Objetivo

Estimar o **efeito causal** da **liquidez extraordinária do comprador** sobre o **prêmio de
reinvestimento**, neutralizando confundidores estruturais de alta dimensão. Responde de fato à
pergunta: *existe um prêmio do vendedor causal — ou é só correlação de clubes ricos?*

## 2. Definições

- **Tratamento (D):** `D = ln(1 + revenue_sales)` — receita de vendas do clube na temporada.
  > A proposta pedia o volume de vendas dos **30 dias anteriores** à compra. A base agrega por
  > **temporada**, não por dia → o tratamento é a liquidez **sazonal**. Isso dilui o efeito e
  > torna a estimativa **conservadora** (ver [pendencias.md](pendencias.md)).
- **Resultado (Y):** `premio_reinvestimento` — o resíduo hedônico da Etapa 1.
- **Confundidores (W):** **19 variáveis** efetivamente usadas no notebook:
  - **rede:** `in_degree`, `out_degree`, `pagerank`
  - **elenco:** `squad_size`, `average_age`, `national_team_players`
  - **temporada:** 7 dummies (`season_2018` … `season_2025`)
  - **liga:** 6 dummies de `competition_code`
  > ⚠️ **Atenção:** ao contrário do que a spec (docx) sugeria, o W **não inclui** o bloco
  > financeiro direto (`total_spend`, `n_buys`, `net_balance`, `net_transfer_record`) nem
  > `in_strength`/`out_strength`/`net_flow`/`log_league_mv`. O confundidor **C1 ("clube rico")** é
  > controlado **por proxies** (centralidade na rede + tamanho de elenco + dummies de liga), não pelo
  > volume financeiro direto — uma limitação a registrar.

## 3. Por que Double ML?

Queremos o efeito causal `θ₀` de D em Y, mas confundidores W afetam ambos. Regredir `Y ~ D + W`
diretamente viesaria θ₀ por dois motivos: (1) a relação entre W e Y pode ser **não-linear**, e
(2) há **muitos** confundidores. O DML (Chernozhukov et al., 2018) resolve isso com o modelo
parcialmente linear:

$$Y = \theta_0 D + g(W) + U, \qquad D = m(W) + V$$

Três pilares:

1. **Ortogonalização de Neyman.** θ vem da regressão dos resíduos: `Ỹ = Y − ĝ(W)` contra
   `D̃ = D − m̂(W)`. Erros de 1ª ordem em ĝ e m̂ não viesam θ̂.
2. **Cross-fitting (K=5).** ĝ e m̂ são treinados em K−1 partições e avaliados na restante, em
   rodízio → elimina viés de sobreajuste.
3. **Inferência robusta.** Erro-padrão **heterocedástico-robusto (HC1)** sobre a regressão final
   dos resíduos. *(Obs.: a spec mencionava clusterização por clube, mas a implementação — tanto a
   original em econml quanto a portada — usa SE robusto não-clusterizado.)*

Implementação atual: **Double ML manual** (cross-fitting + partialling-out), pois o econml não
instala no ambiente; modelos de 1ª etapa = Random Forest. CATE via **R-learner**.

## 4. Resultado principal — ATE (efeito causal médio)

| Métrica | 3 temporadas (antigo) | **8 temporadas (atual)** |
|---------|------------------------|--------------------------|
| **θ₀ (ATE)** | 0,0118 | **0,0044** |
| IC 95% (HC1) | [0,0003; 0,0232] | **[−0,0016; 0,0104]** |
| IC 95% (cluster clube×temporada) | — | **[−0,0021; 0,0110]** |
| Significância | Sim, a 5% (no limiar) | **NÃO** (HC1 e cluster) |

**Interpretação (atual):** na média de 8 temporadas, **não há efeito causal significativo** — o IC
cruza zero (com HC1 e com SE clusterizado) e o placebo embaralhado (0,0024) fica quase do tamanho
do ATE. O efeito significativo da versão de 3 temporadas era, em boa parte, **reflexo do período
2022–2023** (mercado aquecido pós-COVID). Ver θ por temporada na seção 8 e o
[changelog](reprocessamento_8temporadas.md).

## 5. Validação

### Placebos (8 temporadas)
| Placebo | θ estimado | IC 95% |
|---------|-----------|--------|
| D embaralhado (shuffle) | 0,0024 | cruza zero ✔ |
| D → ruído gaussiano | 0,0009 | cruza zero ✔ |

Ambos centrados em zero → o modelo **não inventa efeito** onde não há. Note que o shuffle (0,0024)
está **quase do tamanho do ATE (0,0044)** — coerente com o ATE ser, na média, **nulo**.

### Diagnóstico de primeira etapa (8 temporadas)
- **R² model_t (m̂) = 0,37** — confundidores preveem moderadamente a receita.
- **R² model_y (ĝ) = 0,06** — baixo, esperado (Y já é resíduo; atributos do jogador já saíram).

## 6. Heterogeneidade — CATE (R-learner; 8 temporadas)

Estima um efeito causal **por observação** (X = W). *Antes via `CausalForestDML`; agora via
**R-learner** porque o econml não instala — números não diretamente comparáveis.*

| Métrica | Valor |
|---------|-------|
| média dos CATEs | 0,010 (≈ ATE, como esperado) |
| desvio-padrão | 0,066 |
| faixa | **[−0,39; +2,25]** |

**Heterogeneidade substancial:** há clubes com efeito muito acima da média e clubes com efeito
**negativo** (mais liquidez → menor prêmio). **É aqui que mora a história do projeto** — não na
média, mas na dispersão. ⚠️ A cauda superior (máx +2,25) é dominada por **um outlier** (Nîmes,
poucas obs), o que comprime a normalização do IVB — tratar o ranking como **ilustrativo**.

## 7. IVB — Índice de Vulnerabilidade de Barganha

Normaliza o CATE de cada clube no intervalo [0, 1]:

$$IVB_c = \frac{\theta_c - \min(\Theta)}{\max(\Theta) - \min(\Theta)}$$

- **IVB ≈ 1 — "presa fácil":** paga os maiores sobrepreços quando chega capitalizado.
- **IVB ≈ 0 — "negociador disciplinado":** imune à pressão da liquidez.

Filtro de **≥ 3 transações** para estabilidade. Saída: `resultados_etapa2_ivb.csv` (190 clubes,
colunas `buyer, theta_medio, ivb_medio, n_transacoes, ivb_std, rank`).

Ranking atual (8 temporadas, R-learner) — *ilustrativo*:

| "Presas fáceis" (IVB alto) | "Negociadores disciplinados" (IVB baixo) |
|----------------------------|------------------------------------------|
| Nîmes Olympique — 0,52 | SD Eibar — 0,09 |
| KV Mechelen — 0,18 | Hellas Verona — 0,11 |
| Sunderland AFC — 0,18 | Arminia Bielefeld — 0,12 |

> ⚠️ O ranking **mudou completamente** vs. a versão de 3 temporadas (antes FC Arouca/Elche/Sporting
> no topo) — efeito combinado de mais dados **e** da troca de método (R-learner vs. CausalForestDML).
> O topo (Nîmes, 0,52, 8 obs) é um **outlier**. Tratar como **ilustrativo**.

**Por liga:** com os dados novos a diferença entre ligas é **desprezível** (IVB médio ~0,126–0,138
para todas). O padrão antigo ("La Liga/Ligue 1 mais disciplinadas") **não se sustentou** — aliás, a
Ligue 1 aparece agora como a *menos* disciplinada. **Não usar esse corte por liga** na narrativa.

> A mesma lógica, do lado vendedor, identificaria os **"clubes predadores"** — mas a implementação
> atual é **só do lado comprador** (ver [pendencias.md](pendencias.md)).

## 8. Estabilidade temporal — onde o efeito vive (atualizado para 8 temporadas)

Sem datas diárias não dá para estimar a curva de decaimento θ(Δt) proposta. Como substituto,
estima-se θ por temporada. **Com 8 temporadas, fica claro que o efeito NÃO é estável** — ele se
concentra nos anos de mercado aquecido:

(IC clusterizado por clube; `p_bonf` = p após Bonferroni nos 8 testes)

| Temporada | θ | IC 95% (cluster) | Significativo? | p_bonf |
|-----------|-----|------------------|----------------|--------|
| 2017 | −0,006 | [−0,026; 0,014] | não | 1,00 |
| 2018 | −0,013 | [−0,036; 0,011] | não | 1,00 |
| 2019 | +0,024 | [−0,011; 0,058] | não | 1,00 |
| 2021 | +0,012 | [−0,001; 0,025] | não | 0,63 |
| **2022** | **+0,064** | [0,028; 0,101] | **SIM** | **0,005** |
| **2023** | **+0,047** | [0,017; 0,077] | **SIM** | **0,019** |
| 2024 | +0,003 | [−0,053; 0,059] | não | 1,00 |
| 2025 | +0,015 | [−0,010; 0,040] | não | 1,00 |

➡️ O prêmio do vendedor emerge em **2022–2023** (boom pós-COVID) e desaparece nos demais anos.
Esse é o achado central do reprocessamento. A análise antiga (2023–2025) parecia significativa por
"pegar" o 2023 forte.

### Robustez — teste de tratamento defasado (C2)
Implementado em `etapa2_robustez.ipynb`. Usando a liquidez **da temporada anterior** (predeterminada
em relação à compra), com SE clusterizado: θ_defasado = **0,0051\***, IC [0,0012; 0,0090] (n=4.008).
Isso **enfraquece a causalidade reversa (C2)**, mas **não é conclusivo**: na mesma subamostra o
contemporâneo é n.s., sugerindo que o lag capta um traço persistente de clube (corr=0,44), não um
efeito limpo. Reportado abertamente.

## 9. Limitações reconhecidas

1. **Variáveis ausentes:** sem receita de patrocínio, classificação para UCL, dias na janela.
2. **Granularidade temporal:** D reflete a temporada inteira, não 30 dias → efeito diluído.
3. **Controle de C1 por proxies:** o W não tem o volume financeiro direto (ver §2).
4. **C3 (causa comum)** não é controlado. **C2 (causalidade reversa)** foi **testada** via
   tratamento defasado (§8) — efeito persiste, o que a enfraquece.
5. **CATE/IVB via R-learner** (econml indisponível) — aproximação do CausalForestDML.

> Nota: com a base de 8 temporadas reconsolidada, a **imputação de variáveis de elenco caiu para 0
> nulos** (a limitação antiga de "706/750 obs de 2025 imputadas" **deixou de existir**).

A lista de pendências priorizadas está em [pendencias.md](pendencias.md).

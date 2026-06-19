# Etapa 2 — Estimação Causal por Aprendizado de Máquina Duplo

> **Status: documento alinhado à implementação final** (`notebooks/etapa2_double_ml.ipynb` e
> `etapa2_robustez.ipynb`), sobre `transfers_etapa2_ready.csv`
> (**5.146 transferências · 8 temporadas (2017–2025, exceto 2020) · 7 ligas**).
> Substitui a "Etapa 2" da proposta original, que dependia de variáveis indisponíveis na base
> (data exata da transferência, contexto esportivo, performance individual).
>
> **Decisões da implementação que diferem da proposta:** (1) o `econml` não instala no ambiente
> (Python 3.12 + NumPy 2.x) → usamos **DML manual** (Robinson + cross-fitting + HC1 **e SE
> clusterizado clube×temporada**) e **R-learner** no lugar do `CausalForestDML`; (2) o vetor `W`
> **NÃO** inclui o bloco financeiro direto (`total_spend`, `n_buys`, `net_balance`) — o confundidor
> C1 é controlado por *proxies* de rede e elenco (decisão para evitar *bad control*, já que esses
> agregados são parcialmente mediadores/colliders); (3) o resíduo Y da Etapa 1 é **out-of-fold**.

## 1. Da Etapa 1 para a Etapa 2

A Etapa 1 entregou o **resíduo hedônico** de cada transferência:

$$Y_{i,c} = \ln(P_i) - \ln(\hat{P}_i)$$

onde $\hat{P}_i$ é o preço justo previsto pelo modelo Random Forest treinado
apenas com atributos do jogador e controles estruturais de mercado
(idade, idade², log do market value, posição, dummies de liga e temporada).

Por construção, $Y_{i,c}$ está livre de características intrínsecas do atleta.
O que sobra no resíduo é, por hipótese, **forças conjunturais da negociação**.
A Etapa 2 testa se uma dessas forças é a *liquidez recente do comprador*.

## 2. Definição operacional de tratamento

A proposta original definia o tratamento como o logaritmo do volume acumulado
de vendas do clube nos **30 dias anteriores** à compra. A base disponível,
porém, agrega transações por **temporada**, não por dia. Operacionalizamos
o tratamento na granularidade efetivamente disponível:

$$D_c = \ln(1 + \text{revenue\_sales}_{c,t})$$

onde $\text{revenue\_sales}_{c,t}$ é a soma de fees recebidos pelo clube $c$
em vendas durante a temporada $t$. Variantes binárias (`big_sale_flag` =
1 se $\max\_\text{sale}_c > €30M$) podem ser testadas como robustez para
isolar o efeito de vendas *blockbuster*.

## 3. Especificação do modelo Double ML

Seguimos o modelo parcialmente linear de Chernozhukov et al. (2018):

$$Y = \theta D + g(W) + U, \quad \mathbb{E}[U \mid W, D] = 0$$
$$D = m(W) + V, \quad \mathbb{E}[V \mid W] = 0$$

onde $W$ é o vetor de confundidores estruturais do comprador, da liga e da
temporada. A escolha de $W$ é direta dado o que o dataset oferece e está
detalhada na Tabela 1.

**Vetor $W$ efetivamente usado (19 variáveis):**

| Bloco | Variáveis em $W$ | Confundidor neutralizado |
|---|---|---|
| Estrutural do elenco | `squad_size`, `average_age`, `national_team_players` | C1, C6 — perfil/prestígio |
| Posição na rede | `pagerank`, `in_degree`, `out_degree` | C1, C6 — poder de barganha estrutural |
| Liga | 6 dummies `competition_code_*` | C4, C5 — poder econômico da liga |
| Temporada | 7 dummies `season_*` (base: 2017) | C4, C5 — sazonalidade e inflação |

**Importante (decisão de desenho):** `W` **não** inclui o bloco financeiro direto (`total_spend`,
`n_buys`, `net_balance`, `net_transfer_record`) nem `in_strength`/`out_strength`/`net_flow`/`log_league_mv`.
Esses agregados são contemporâneos e parcialmente **mediadores/colliders** (p.ex. `net_balance`
contém o próprio `revenue_sales`); incluí-los seria *bad control*. O confundidor C1 ("clube rico")
é, portanto, controlado por **proxies** de centralidade na rede (PageRank, graus) e porte de elenco —
uma aproximação reconhecida nas limitações, não o controle financeiro direto ideal.

### Confundidores ainda não controlados

| ID | Bias | Estratégia futura |
|---|---|---|
| C2 | Causalidade reversa (compra precede venda, multa rescisória) | Enriquecer com flag de multa via scraping do Transfermarkt |
| C3 | Causa comum (classificação para UCL, troca de técnico) | Integrar dataset de campeonatos e calendário esportivo |

Esses dois confundidores permanecem como limitação reconhecida do desenho
atual e devem ser abordados na próxima iteração da modelagem.

## 4. Protocolo algorítmico

A estimação segue os três pilares do DML:

1. **Ortogonalização de Neyman.** O parâmetro causal $\theta$ é obtido pela
   regressão linear simples do resíduo $\tilde{Y} = Y - \hat{g}(W)$ contra
   o resíduo $\tilde{D} = D - \hat{m}(W)$. A propriedade de ortogonalidade
   garante que erros de primeira ordem em $\hat{g}$ e $\hat{m}$ não viciem
   $\hat{\theta}$.
2. **Cross-fitting.** A amostra é dividida em $K=5$ partições; $\hat{g}$ e
   $\hat{m}$ são treinados em $K-1$ partições e os resíduos são avaliados
   na partição restante, em rodízio. Isso elimina o viés de sobreajuste
   típico do reuso de dados.
3. **Inferência robusta.** Erros-padrão são clusterizados por clube comprador
   para acomodar correlação intra-clube ao longo das temporadas.

### Implementação em Python (DML manual, sem econml)

```python
import re
import numpy as np
import pandas as pd
from scipy import stats
from sklearn.ensemble import RandomForestRegressor
from sklearn.model_selection import KFold

df = pd.read_csv('output/transfers_etapa2_ready.csv')

Y = df['premio_reinvestimento'].clip(*df['premio_reinvestimento'].quantile([.01, .99])).values
D = df['log_revenue'].values

# W efetivo: rede + elenco + dummies de liga e temporada (19 vars). SEM bloco financeiro.
structural = ['in_degree', 'out_degree', 'pagerank',
              'squad_size', 'average_age', 'national_team_players']
seasons = sorted(c for c in df.columns if re.fullmatch(r'season_\d{4}', c))
ligas   = [c for c in df.columns if c.startswith('competition_code_')]
W = df[structural + seasons + ligas].values

# Cluster clube x temporada (D/W constantes dentro do grupo).
cluster = (df['buyer'].astype(str) + '_' + df['season_id'].astype(str)).values

def dml(Y, D, W, cluster, K=5, seed=42):
    n = len(Y); yhat = np.zeros(n); dhat = np.zeros(n)
    for tr, te in KFold(K, shuffle=True, random_state=seed).split(W):
        RF = lambda: RandomForestRegressor(200, max_depth=5, random_state=seed, n_jobs=-1)
        yhat[te] = RF().fit(W[tr], Y[tr]).predict(W[te])
        dhat[te] = RF().fit(W[tr], D[tr]).predict(W[te])
    yr, dr = Y - yhat, D - dhat
    X = np.column_stack([np.ones(n), dr]); inv = np.linalg.inv(X.T @ X)
    b = inv @ (X.T @ yr); u = yr - X @ b; theta = b[1]
    meat = np.zeros((2, 2))                                  # SE clusterizado (CR1)
    for c in np.unique(cluster):
        m = cluster == c; sg = X[m].T @ u[m]; meat += np.outer(sg, sg)
    G = len(np.unique(cluster)); cov = inv @ meat @ inv * (G/(G-1)) * ((n-1)/(n-2))
    se = np.sqrt(cov[1, 1])
    return theta, (theta - 1.96*se, theta + 1.96*se), 2*stats.norm.sf(abs(theta/se))

theta, ci, p = dml(Y, D, W, cluster)
print(f'theta={theta:.4f}  IC95%={ci}  p={p:.4f}')
```

A saída fornece $\hat{\theta}$, IC 95 % (clusterizado clube×temporada) e p-valor para
$H_0: \theta = 0$. O CATE/IVB usa **R-learner** (Nie & Wager) no lugar do `CausalForestDML`.

## 5. Achado central e heterogeneidade

### 5.1 Efeito médio (ATE) e o achado por temporada

O ATE estimado é **θ = 0,0044** (IC 95 % HC1 [−0,0016; 0,0104]; IC clusterizado clube×temporada
[−0,0021; 0,0110]) — **não significativo**. A progressão OLS-ingênuo (0,0142\*) → OLS+W (0,0051) →
DML (0,0044) mostra que a correlação bruta era majoritariamente o confundidor de porte ("clube rico").

Estimando θ **por temporada** (IC clusterizado por clube + correção de múltiplas comparações), o
efeito é significativo **apenas em 2022 (+0,064) e 2023 (+0,047)** — o boom pós-COVID — e **sobrevive
a Bonferroni e FDR**. Este é o achado confirmatório: o prêmio do vendedor não é universal, emerge em
regime de mercado aquecido. (Validações na seção 6.)

### 5.2 Índice de Vulnerabilidade de Barganha (IVB) — apêndice exploratório

Com o **R-learner** (substituto do `CausalForestDML`) obtemos um CATE por observação, agregado por
clube (≥ 3 transações). O IVB usa **normalização por percentil** do CATE médio do clube — imune ao
outlier que dominava a versão min-max:

$$IVB_c = \text{percentil}_c(\overline{\theta}_c)$$

> ⚠️ Como o R² de 1ª etapa em Y é só 0,06, há pouco sinal de heterogeneidade genuína. O IVB é
> **apêndice ilustrativo / prova de conceito**, não ferramenta de decisão.

## 6. Testes de robustez (executados)

Implementados em `etapa2_robustez.ipynb` e `etapa2_double_ml.ipynb`:

1. **Especificação de $D$.** D₁=`log_revenue` (n.s.), D₂=`log(n_sales)`, D₃=`big_sale`. D₂/D₃ são
   significativos no bruto, mas **perdem significância** ao controlar intensidade de janela
   (`n_buys`+gasto) → o "mecanismo de sinalização" é **exploratório, não confirmatório**.
2. **Subgrupos por tier de liga.** Top-4 vs. ligas menores — nenhum significativo na média (cluster).
3. **Placebos globais.** D embaralhado (θ≈0,002) e ruído gaussiano (θ≈0,001) — ICs cruzam zero.
4. **Placebo dentro da temporada.** Permuta D entre clubes de 2022/2023 (200×): θ observado fica fora
   do nulo, **p empírico = 0,005** — o efeito sazonal não é artefato do desenho.
5. **Causalidade reversa (C2).** Tratamento defasado t−1 (θ=0,0051\*, cluster) — enfraquece C2, mas
   **não conclusivo** (o contemporâneo é n.s. na mesma subamostra → o lag pode captar traço de clube).
6. **Sensibilidade ao corte de fee (C6).** Pipeline reconstruído a €0/100k/250k/500k/1M — ATE nulo em
   todos; 2022/2023 estáveis de €0 a €500k. Achados não dependem do corte.
7. **Sensibilidade à identificação.** Robustness Value (Cinelli-Hazlett) ≈ 0,24–0,27 para 2022/2023;
   stress test com financeiros contemporâneos (over-control) — 2023 sobrevive, 2022 atenua.

**PSM não foi executado** (substituído pelo controle não-paramétrico via DML) — ver seção 7.

## 7. Limitações reconhecidas

1. **Identificação por *selection-on-observables*.** O confundidor C1 ("clube rico") é controlado por
   *proxies* de rede/elenco, sem o bloco financeiro direto (que seria *bad control*). Com R²_y=0,06,
   isso não é verificável; o Robustness Value (0,24–0,27) mitiga, mas não substitui um instrumento.
2. **Granularidade temporal.** D agrega a temporada inteira, não os 30 dias pré-compra — diluindo o
   efeito (estimativa conservadora) e impedindo a curva de decaimento θ(Δt) da proposta original.
3. **C3 (causa comum)** não controlado (sem dados de UCL, troca de técnico, receita de TV).
4. **CATE/IVB via R-learner** (econml não instala) — aproximação, tratada como apêndice.
5. **Não entregues** (trabalho futuro): PSM ("clubes gêmeos"), integração FBref/StatsBomb,
   `CausalForestDML`, análise do lado vendedor ("clubes predadores").

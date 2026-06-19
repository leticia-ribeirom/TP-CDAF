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

## 5. Métricas inovadoras adaptadas

A proposta original sugeria duas métricas: a Curva de Decaimento Temporal e
o Índice de Vulnerabilidade de Barganha. A primeira é inviável na base atual
porque exige a data exata da transferência. Mantemos a segunda integralmente
e propomos uma terceira métrica, originalmente nossa, que aproveita a
disponibilidade das features de rede.

### 5.1 Índice de Vulnerabilidade de Barganha (IVB)

Estimando efeitos heterogêneos com o **R-learner** (substituto do `CausalForestDML`), obtemos
$\theta_c$ para cada clube comprador (com ≥ 3 transações). O IVB normaliza esse efeito no
intervalo $[0,1]$ (tratado como **exploratório/ilustrativo** — dominado por outlier; ver notebook):

$$IVB_c = \frac{\theta_c - \min(\Theta)}{\max(\Theta) - \min(\Theta)}$$

Clubes com IVB alto são "presas fáceis" — pagam sobrepreços maiores quando
chegam ao mercado recém-capitalizados. Clubes com IVB baixo são negociadores
disciplinados, capazes de reciclar capital sem ceder à pressão.

A mesma lógica, aplicada do lado vendedor, identifica os "clubes predadores"
— times que extraem as maiores taxas de prêmio positivo quando abordados
por compradores recém-capitalizados.

### 5.2 Sensibilidade do prêmio à centralidade na rede (substitui a Curva θ(Δt))

Sem datas diárias, não podemos modelar o decaimento temporal proposto. Em
contrapartida, exploramos uma dimensão ortogonal que a base permite: a
**heterogeneidade do efeito segundo a posição estrutural do comprador na
rede de transferências**. Parametrizamos:

$$\theta(\text{PageRank}_c) = \theta_0 + \theta_1 \cdot \text{quartil}(\text{PageRank}_c)$$

A hipótese testada é que clubes centrais (alta centralidade de PageRank,
top-25 % do mercado) sofrem prêmios menores que clubes periféricos, por
deterem informação superior e poder de barganha estrutural. Essa
parametrização entrega um insight gerencial direto: *quanto vale, em
euros, estar posicionado no núcleo da rede de transferências*.

### 5.3 Curva temporal por temporada (proxy do decaimento)

Como degradação aceitável da curva $\theta(\Delta t)$ original, estimamos
$\theta_t$ separadamente para cada temporada $t \in \{2017, \dots, 2025\}$ (exceto 2020).
O resultado é o **achado central**: o efeito é significativo apenas em **2022 e 2023** (boom
pós-COVID), sobrevivendo a Bonferroni/FDR; nas demais temporadas é indistinguível de zero.

## 6. Testes de robustez planejados

1. **Especificação de $D$.** Replicar com $D = $ `log_revenue`, com
   `n_sales` e com a flag binária `big_sale`.
2. **Subgrupos.** Estimar $\theta$ por tier de comprador (top-5 ligas vs.
   demais) e por posição do jogador.
3. **Placebo.** Aleatorizar $D$ entre clubes mantendo a estrutura de $W$;
   $\theta$ estimado deve ser estatisticamente indistinguível de zero.
4. **Reverse causation.** Reestimar com tratamento defasado (lag de uma
   temporada) — se o efeito persistir, a hipótese de causalidade reversa
   (C2) enfraquece.
5. **PSM como sanity check do C6.** Construir pares de clubes com
   propensity score similar (alto vs. baixo `log_revenue`) e comparar o
   prêmio médio diretamente, sem o ferramental DML.

## 7. Cronograma de implementação

| Semana | Entrega |
|---|---|
| 1 | Pipeline DML completo (LinearDML + CausalForestDML), análises de robustez 1 e 3 |
| 2 | Cálculo de IVB por clube; ranking de clubes-presa e clubes-predadores |
| 2 | Curva $\theta_t$ por temporada + sensibilidade ao PageRank |
| 3 | Validação por PSM e roadmap de enriquecimento (data exata, performance) |

## 8. Limitações reconhecidas

A modelagem atual não controla por contexto esportivo individual
(classificação para a UCL, troca de técnico, choques de receita de TV) nem
isola transferências motivadas por cláusula de multa rescisória. Esses
elementos correspondem aos confundidores C2 e C3 e ficam endereçados como
trabalho futuro. A granularidade temporal trimestral (por temporada) é a
restrição mais relevante: ela limita a precisão da estimativa de urgência
e impede a estimação direta da curva exponencial de decaimento. Ainda
assim, o desenho proposto é suficiente para responder à pergunta principal
do projeto — *existe um prêmio causal do vendedor?* — com rigor
metodológico compatível com o estado da arte.

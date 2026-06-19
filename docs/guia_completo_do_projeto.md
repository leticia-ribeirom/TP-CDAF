# Guia Completo do Projeto — Efeito Dominó no Mercado de Transferências
### Grupo 02 · Ciência de Dados Aplicada ao Futebol · UFMG · 2026

> **Para quem é este documento:** todos os integrantes do grupo, especialmente quem não acompanhou
> cada etapa em tempo real. Leia do início ao fim antes da apresentação. O objetivo é que qualquer
> um de nós consiga explicar qualquer parte do projeto com confiança.

---

## Índice

1. [A Pergunta e a Hipótese](#1-a-pergunta-e-a-hipótese)
2. [Os Dados](#2-os-dados)
3. [Feature Engineering](#3-feature-engineering)
4. [Etapa 1 — Modelo Hedônico](#4-etapa-1--modelo-hedônico)
5. [Etapa 2 — Double Machine Learning](#5-etapa-2--double-machine-learning)
6. [Testes de Robustez](#6-testes-de-robustez)
7. [IVB — Índice de Vulnerabilidade de Barganha](#7-ivb--índice-de-vulnerabilidade-de-barganha)
8. [Os Três Achados Principais](#8-os-três-achados-principais)
9. [O Que Não Fizemos e Por Quê](#9-o-que-não-fizemos-e-por-quê)
10. [Glossário Rápido](#10-glossário-rápido)

---

## 1. A Pergunta e a Hipótese

### O que queríamos saber

Quando um clube de futebol faz uma grande venda — digamos, vende seu melhor jogador por €50M —
ele fica com dinheiro em caixa e precisa repor o atleta. Os clubes vendedores sabem disso. A
intuição do projeto é: **esse clube fica em desvantagem na negociação e acaba pagando mais caro
do que deveria nas compras seguintes?**

Chamamos esse efeito de **"prêmio do vendedor"** — um sobrepreço pago pelo clube que acabou de
vender.

### Por que isso não é óbvio

A primeira resposta de qualquer pessoa seria: "é claro que existe — olha o Real Madrid, vende
caro e compra caro também". **Mas isso é o problema.** Clubes grandes vendem por valores altos E
compram por valores altos — não por causa do efeito de barganha, mas simplesmente porque são
ricos. Se pegarmos todos os dados e fizermos uma correlação, vamos encontrar uma relação positiva
entre "quanto vendeu" e "quanto pagou de sobrepreço" — mas essa correlação é falsa. É o clube
rico aparecendo dos dois lados.

**O desafio real do projeto é separar o efeito causal da liquidez do simples porte financeiro do
clube.** Para isso, precisamos de uma estratégia metodológica cuidadosa — não podemos responder
essa pergunta com uma regressão simples.

### A pergunta operacional

Depois de refinar, nossa pergunta virou:

> **A liquidez recente do comprador (receita de vendas na temporada) tem efeito causal sobre
> o sobrepreço que ele paga em novas contratações, depois de neutralizar os confundidores
> estruturais?**

Duas palavras importantes: **causal** (não correlação) e **depois de neutralizar confundidores**
(controlando o clube rico e outros fatores).

### Os 6 Confundidores Mapeados

Antes de modelar qualquer coisa, mapeamos 6 razões pelas quais poderíamos encontrar uma correlação
espúria entre "recebeu de vendas" e "pagou sobrepreço":

| ID | Confundidor | O que é | Como tratamos |
|----|-------------|---------|---------------|
| C1 | Clube rico | Clubes grandes vendem caro E compram caro | Controles de rede (PageRank) + elenco no vetor W |
| C2 | Causalidade reversa | O clube planejou comprar caro e vendeu pra financiar | Teste de tratamento defasado (lag t-1) |
| C3 | Causa comum | UCL, troca de técnico afeta vendas e compras | **Não controlado** (sem dados esportivos) |
| C4 | Sazonalidade | Certas janelas são mais caras por natureza | Dummies de temporada em W e na Etapa 1 |
| C5 | Inflação de mercado | O mercado de 2023 é mais caro que 2017 | Dummies de temporada absorvem inflação |
| C6 | Viés de seleção | Clubes diferentes têm perfis de compra diferentes | Proxies de rede + tamanho de elenco em W |

O fato de termos mapeado esses confundidores explicitamente **antes** de modelar é o que dá
credibilidade metodológica ao trabalho. Não estamos ignorando os problemas — estamos tratando
cada um deles.

---

## 2. Os Dados

### Fonte

Usamos o **Transfermarkt** — o maior banco de dados de transferências de futebol do mundo —
via o repositório público `dcaribou/transfermarkt-datasets`. Os dados incluem:

- Todas as transferências de 7 ligas europeias (Premier League, La Liga, Bundesliga, Serie A,
  Ligue 1, Liga Portugal, Jupiler Pro League)
- Informações dos jogadores: idade, posição, valor de mercado
- Informações dos clubes: elenco, ligas, histórico de transações

### O funil de filtragem

Começamos com **44.627 movimentações brutas** e terminamos com **5.146 transferências** para
modelagem. Por quê tantos filtros?

**Empréstimos e transferências gratuitas foram removidos** porque não têm preço negociado —
não faz sentido estudar sobrepreço onde não há preço.

**Transfers abaixo de €250.000 foram removidos** por uma razão empiricamente validada: abaixo
desse valor, o indicador de sobrepreço tem variância muito baixa (desvio padrão < 0,30) e
mediana fortemente negativa. Não há dinâmica real de mercado nessa faixa — são transações
administrativas, muitas vezes jogadores saindo de graça mas com taxa de trâmite simbólica.

**Transferências sem valor de mercado válido foram removidas** porque o valor de mercado do
Transfermarkt é a base do nosso "preço justo" na Etapa 1. Sem esse dado, não conseguimos calcular
o resíduo.

### As 8 Temporadas

O projeto começou com 3 temporadas (2023–2025) e foi expandido para **8 temporadas
(2017–2025, exceto 2020)**. Por quê?

- Mais dados = estimativas mais precisas e ICs menores
- 8 temporadas permitem ver variação temporal real (boom pós-COVID em 2022–2023)
- 2020 está ausente porque a pandemia paralisou os mercados — os dados não foram coletados

**Consequência importante:** a expansão para 8 temporadas **mudou o resultado principal**.
Com 3 temporadas, o ATE era significativo (θ = 0,012). Com 8 temporadas, o ATE médio ficou
**não significativo** (θ = 0,0044). Isso não é uma falha — é um resultado mais honesto e mais
rico. Explicamos por quê na seção 5.

---

## 3. Feature Engineering

### O que é feature engineering

É o processo de criar as variáveis que vão entrar nos modelos a partir dos dados brutos. No nosso
caso, os dados brutos são tabelas de transferências e clubes — precisamos transformá-los em
variáveis numéricas que capturem os conceitos que nos interessam.

### Features do jogador (usadas na Etapa 1)

Essas variáveis descrevem o atleta e o contexto de mercado. Entram no modelo hedônico.

- **`age` e `age_sq`:** idade e idade ao quadrado. O quadrado é essencial porque a relação
  entre idade e valor não é linear — jogadores com ~20 anos valem mais que a curva linear
  sugeriria (potencial de crescimento), e depois dos 30 o valor cai mais rápido. Sem o termo
  quadrático, o modelo subestimaria o valor de jogadores jovens e superestimaria os veteranos.

- **`log_mv`:** logaritmo do valor de mercado do Transfermarkt. É o preditor mais forte — o
  mercado já "precificou" o jogador antes da transferência. Usamos logaritmo porque os valores
  variam de €250k a €200M — em escala linear, os outliers dominariam o modelo.

- **`is_attacker`, `is_midfielder`, `is_defender`:** dummies de posição. Atacantes comandam
  prêmios maiores em média. Base de comparação: goleiros e zagueiros.

- **Dummies de liga (6):** Premier League, La Liga, Bundesliga (base), Serie A, Ligue 1,
  Liga Portugal, Jupiler Pro League. Ligas mais ricas têm inflação de preços própria.

- **Dummies de temporada (7):** 2018, 2019, 2021, 2022, 2023, 2024, 2025 (base: 2017).
  Capturam a inflação geral do mercado ao longo do tempo.

**Total: 19 features.** Nenhuma feature de comportamento financeiro do clube comprador entra
aqui — isso é um design intencional, explicado na Etapa 1.

### Features do clube (usadas na Etapa 2)

Essas variáveis descrevem o comportamento do clube na temporada. Entram como confundidores (W)
no Double ML.

- **`revenue_sales`:** soma de todos os fees recebidos em vendas. É a nossa **variável de
  tratamento principal (D)** — transforma-se em `D = log(1 + revenue_sales)`.

- **`n_sales`:** número de vendas realizadas. Alternativa de tratamento testada na robustez.

- **`max_sale`:** maior venda individual. Base para criar `big_sale_flag` (>€30M).

- **`total_spend`, `n_buys`:** controles do comportamento habitual de gasto do clube.
  Importantes para separar "teve dinheiro esta temporada" de "sempre gasta muito".

### Features de rede (usadas na Etapa 2)

O mercado de transferências pode ser modelado como um **grafo dirigido**: nós são clubes, arestas
são transferências, pesos são os fees pagos. Para cada temporada, construímos esse grafo e
calculamos:

- **`pagerank`:** importância estrutural do clube na rede. Clubes que compram de outros clubes
  importantes têm PageRank alto. É a proxy mais robusta de "poder e prestígio" do clube.

- **`in_degree`:** quantos clubes diferentes venderam jogadores para este clube — mede a
  diversidade de parceiros de compra.

- **`out_degree`:** quantos clubes diferentes compraram deste clube — mede o alcance como
  vendedor.

- **`in_strength` / `out_strength`:** volume total (em €) de compras e vendas. O `out_strength`
  e `in_strength` capturam o mesmo conceito que `revenue_sales` mas pela perspectiva da rede.

Essas features de rede são importantes porque capturam o **poder estrutural de barganha** de um
clube — algo que simples variáveis financeiras não capturam. O PSG e o Real Madrid dominam o
PageRank; clubes menores têm PageRank baixo. Isso controla o confundidor C1 (clube rico) de
forma mais sofisticada que uma dummy de liga.

---

## 4. Etapa 1 — Modelo Hedônico

### O que é um modelo hedônico

Um "modelo hedônico" decompõe o preço de um bem na soma do valor de seus atributos. É usado em
imóveis ("preço do apartamento = f(metros quadrados, localização, andar...)"), em carros, em
vinhos. Aqui, aplicamos ao futebol:

> **Preço da transferência = f(atributos do jogador + contexto de mercado)**

O que o modelo **não consegue explicar** — o resíduo — é o que sobra depois de remover o valor
intrínseco do atleta. Esse resíduo é o nosso **sobrepreço** ou **prêmio de reinvestimento (Y)**.

### Por que precisamos do modelo hedônico antes do DML

Se tentássemos medir o sobrepreço diretamente comparando o preço pago com o valor de mercado do
Transfermarkt, teríamos um proxy razoável. Mas o value de mercado do Transfermarkt também tem
erro — é uma estimativa humana, não o valor justo de ML. Ao treinar um modelo com 4 algoritmos
e 19 features, conseguimos uma estimativa mais precisa e menos enviesada do "preço justo".

Mais importante: **o modelo hedônico separa o que é do jogador do que é da negociação**. O
resíduo da Etapa 1 está livre de "ele pagou caro porque o jogador era muito bom". O que sobra
são as forças conjunturais da negociação — que é exatamente o que queremos medir na Etapa 2.

### A decisão do split temporal

Treinamos em 2017–2024 e testamos em 2025. **Por que não aleatório?**

Divisão aleatória causaria *data leakage* temporal: o modelo veria dados do futuro no treino.
Em termos práticos, se misturarmos uma transferência de 2023 no teste e outra de 2023 no treino,
o modelo aprende padrões específicos daquele ano — mas na vida real, você sempre vai prever o
futuro a partir do passado. O split temporal simula o uso real e é um teste de generalização
mais honesto.

### Os 4 modelos testados e por que

Testamos 4 algoritmos para ter certeza de que o resultado não depende da escolha do modelo:

| Modelo | Test R² | Test RMSE | Por que testar |
|--------|---------|-----------|----------------|
| **Random Forest** ✅ | **0,762** | **0,673** | Ensemble robusto, sem overfitting por OOB |
| LightGBM | 0,755 | 0,683 | Boosting eficiente, compara com RF |
| XGBoost | 0,746 | 0,694 | Boosting clássico da literatura |
| SVR (RBF) | 0,730 | 0,716 | Kernel não-linear como contraste |

**Vencedor: Random Forest**, por menor RMSE e maior R² no holdout. O gap pequeno entre os
modelos (todos entre 0,73 e 0,76) é positivo — significa que o resultado é robusto à escolha
do algoritmo.

**Por que RMSE como métrica principal?** Porque os fees variam de €250k a €200M em escala
logarítmica. O RMSE penaliza erros grandes — que importam quando estamos falando de dezenas de
milhões de euros.

### O que o R² = 0,762 significa

76,2% da variância do log do fee de transferência é explicada por apenas 19 features do jogador
e do contexto de mercado. É um resultado excelente para essa aplicação — a literatura acadêmica
de valuation de jogadores tipicamente reporta R² entre 0,70 e 0,80.

Os 23,8% não explicados incluem: contexto tático específico do clube comprador, urgência do
negócio, relacionamento entre presidentes, cláusulas de bônus não divulgadas, pressão de torcida.
Parte desse "ruído" é o nosso objeto de estudo na Etapa 2.

### O SHAP e o que ele revela

SHAP (SHapley Additive exPlanations) é uma técnica de interpretabilidade que calcula a
contribuição de cada feature para cada predição. Para o nosso modelo:

1. **`log_mv` domina** — o valor de mercado do Transfermarkt explica a maior parte do preço.
   Isso valida o dado: Transfermarkt precifica bem os jogadores.
2. **`age` e `age_sq`** vêm em seguida — a curvatura biológica do valor é capturada.
3. Dummies de **liga e posição** têm impacto secundário.

Isso é o resultado esperado e dá credibilidade ao modelo. Se uma feature estranha dominasse,
teríamos um problema.

### O resíduo: nossa variável Y

$$Y_i = \ln(P_i) - \ln(\hat{P}_i)$$

- **Y > 0:** clube pagou acima do preço justo → sobrepreço (prêmio)
- **Y = 0:** clube pagou exatamente o valor justo
- **Y < 0:** clube fez um bom negócio — comprou abaixo do valor justo

**52,5% das transferências têm Y > 0** (resíduo out-of-fold) — mais da metade do mercado paga
algum prêmio. A mediana é +3,5%, um leve sobrepreço típico. Mas atenção: isso não significa que
o prêmio do vendedor existe — pode ser simplesmente que o mercado sempre tem um pequeno ágio médio
por incerteza e urgência. O efeito causal da liquidez é testado na Etapa 2.

> **Nota metodológica (correção importante):** o resíduo é gerado **out-of-fold** (`cross_val_predict`),
> não prevendo o próprio dado de treino. Isso evita resíduos in-sample artificialmente pequenos para
> 85% das observações — um vazamento que contaminava a Etapa 2 na versão anterior.

---

## 5. Etapa 2 — Double Machine Learning

### O problema que o DML resolve

Queremos estimar o efeito causal θ de D (liquidez) em Y (sobrepreço). Se fizéssemos só uma
regressão `Y ~ D`, o coeficiente de D seria enviesado pelos confundidores W (clube rico,
sazonalidade, etc.). Se adicionássemos W na regressão (`Y ~ D + W`), o problema seria:

1. A relação entre W e Y pode ser não-linear (Random Forest captura isso, OLS não)
2. Temos 19 confundidores — incluí-los em OLS aumenta o erro e pode criar overfitting

O **Double ML** (Chernozhukov et al., 2018) resolve os dois problemas com uma técnica elegante
chamada **ortogonalização de Neyman**. A ideia em 3 passos:

**Passo 1:** Tire de Y o que os confundidores W explicam → resíduo Ỹ = Y − ĝ(W)

**Passo 2:** Tire de D o que os confundidores W explicam → resíduo D̃ = D − m̂(W)

**Passo 3:** Estime θ pela regressão simples Ỹ ~ θ · D̃

**Por que isso funciona?** Ỹ e D̃ são as partes de Y e D que os confundidores *não conseguem
explicar*. Ao regredir Ỹ em D̃, estamos medindo a relação entre "variação de liquidez que não
é explicada pelo porte do clube" e "variação de sobrepreço que não é explicada pelo porte do
clube". Essa relação é, por construção, livre do confundidor C1.

### O cross-fitting e por que é necessário

Se treinarmos ĝ e m̂ nos mesmos dados onde calculamos os resíduos, o modelo pode "memorizar"
os dados de treino e os resíduos ficarão artificialmente pequenos — um tipo de overfitting que
viesaria θ.

**Cross-fitting (K=5 folds):** dividimos os dados em 5 partes. Para cada parte, treinamos ĝ
e m̂ nas outras 4 e calculamos os resíduos na parte deixada de fora. Ao final, temos resíduos
out-of-sample para todas as observações — sem overfitting.

### O vetor de confundidores W (19 variáveis)

| Bloco | Variáveis | Confundidor controlado |
|-------|-----------|------------------------|
| Rede | `in_degree`, `out_degree`, `pagerank` | C1 (clube rico/prestígio), C6 |
| Elenco | `squad_size`, `average_age`, `national_team_players` | C1, C6 |
| Temporada | 7 dummies (2018–2025) | C4 (sazonalidade), C5 (inflação) |
| Liga | 6 dummies | C4, C5 |

**Limitação importante:** W não inclui `total_spend`, `n_buys`, `net_balance` (bloco financeiro
direto). O confundidor C1 é controlado por *proxies* (centralidade na rede + tamanho do elenco),
não pelo volume financeiro direto. Isso é uma limitação reconhecida — declaramos nas limitações.

### O resultado principal: ATE = 0,0044 (não significativo)

**ATE (Average Treatment Effect)** = efeito médio causal. θ = 0,0044 é uma **semielasticidade**
(variação do sobrepreço por unidade-log de receita de vendas — **não** "por 1% de receita").

**O IC 95% cruza zero tanto com HC1 [−0,0016; 0,0104] quanto com erro-padrão clusterizado
clube×temporada [−0,0021; 0,0110].** Não podemos rejeitar a hipótese de que o efeito médio é zero.

> **Por que clusterizar?** D e W são constantes dentro de cada clube×temporada (só 974 clusters
> para 5.146 transferências). Tratar as 5.146 obs como independentes subestima os SE; por isso
> reportamos também o IC clusterizado, que é o nível de inferência correto.

**Por que isso é um resultado válido e não uma falha?**

Porque o contraste com a correlação ingênua revela algo importante:

| Estimador | θ | IC 95% (HC1) | Significativo? |
|-----------|---|--------------|----------------|
| OLS sem controles (`Y ~ D`) | **0,0142** | [0,009; 0,019] | **Sim** |
| OLS com controles lineares | 0,0051 | [−0,000; 0,010] | Não |
| **DML (não-linear)** | **0,0044** | [−0,002; 0,010] | **Não** |

A progressão é clara: à medida que controlamos os confundidores de forma cada vez mais adequada,
o efeito estimado encolhe de 0,0142 para 0,0044 e perde significância. **Isso mostra que a
correlação bruta (0,0142) era, em grande parte, o confundidor "clube rico" em ação.** O DML
separou o efeito real do ruído de confundimento. (A correlação bruta também é significativa com
SE clusterizado; o DML não.)

### Os placebos: validando que o modelo não inventa efeito

Dois testes de sanidade:

**Placebo 1 — D embaralhado:** embaralhamos aleatoriamente quem recebeu qual valor de receita
de vendas. Se o modelo encontrar um efeito aqui, ele está inventando efeito onde não há. Resultado:
θ = 0,0024 (IC cruza zero) ✅

**Placebo 2 — D → ruído gaussiano:** substituímos D por números aleatórios. Resultado:
θ = 0,0009 (IC cruza zero) ✅

**Atenção:** o placebo embaralhado deu θ = 0,0024 — quase do tamanho do ATE real (0,0044).
Isso reforça que o ATE **médio** é genuinamente nulo: o modelo mal distingue o tratamento real do
ruído na amostra completa. (Esse placebo embaralha D globalmente, então testa o efeito médio, não
o poder de detectar os efeitos sazonais de 2022/2023 — ver limitações.)

### O R² da primeira etapa

Antes de estimar θ, o DML precisa que os modelos ĝ e m̂ funcionem minimamente bem:

- **R² model_t (m̂) = 0,37:** os confundidores W explicam 37% da variância da receita de vendas.
  Razoável — o porte do clube prediz razoavelmente bem quanto ele vende.
- **R² model_y (ĝ) = 0,06:** os confundidores W explicam apenas 6% da variância do sobrepreço.
  Baixo, mas esperado: Y já é o resíduo da Etapa 1 (jogador foi removido). O que sobra é
  majoritariamente idiossincrático.

### O achado central: θ por temporada

Quando olhamos o efeito separado por temporada, a história muda completamente:

(θ por temporada, IC clusterizado por clube; `p_bonf` = p-valor após Bonferroni para os 8 testes.)

| Temporada | θ | IC 95% (cluster) | Sig.? | p_bonf | Contexto |
|-----------|---|------------------|-------|--------|----------|
| 2017 | −0,006 | [−0,026; 0,014] | Não | 1,00 | Mercado normal |
| 2018 | −0,013 | [−0,036; 0,011] | Não | 1,00 | Mercado normal |
| 2019 | +0,024 | [−0,011; 0,058] | Não | 1,00 | Mercado normal |
| 2021 | +0,012 | [−0,001; 0,025] | Não | 0,63 | Pós-COVID (cautela) |
| **2022** | **+0,064** | **[0,028; 0,101]** | **Sim** | **0,005** | **Boom pós-COVID** |
| **2023** | **+0,047** | **[0,017; 0,077]** | **Sim** | **0,019** | **Boom pós-COVID** |
| 2024 | +0,003 | [−0,053; 0,059] | Não | 1,00 | Normalização |
| 2025 | +0,015 | [−0,010; 0,040] | Não | 1,00 | Normalização |

**2022 e 2023 sobrevivem à correção de Bonferroni E ao FDR** — não são falsos-positivos do
"garden of forking paths". Este é o achado confirmatório mais forte sobre o efeito sazonal.

**O que isso significa:**

Em 2022 e 2023, o mercado de transferências viveu um boom sem precedentes após 2 anos de
restrições pandêmicas. Os volumes bateram recordes. Nesse contexto de escassez de talento
disponível e urgência generalizada de reposição, o prêmio do vendedor emergiu: clubes com
liquidez pagavam significativamente acima do preço justo (efeito sazonal +0,064 em 2022 e
+0,047 em 2023, ambos sobrevivendo a Bonferroni/FDR).

Nos outros 6 anos, o efeito é estatisticamente zero. A média das 8 temporadas dilui o efeito
de 2022–2023 com 6 anos de zero — daí o ATE médio ser nulo.

**Por que a versão de 3 temporadas parecia significativa?** Porque usava 2023–2025, e 2023 era
o ano mais forte. Com mais dados, o contexto ficou claro: não é sempre — é só quando o mercado
aquece.

---

## 6. Testes de Robustez

### O que são testes de robustez e por que fazemos

Um resultado robusto é aquele que não muda quando fazemos variações razoáveis na metodologia.
Se o efeito desaparece quando mudamos levemente o tratamento ou a amostra, o resultado original
não era confiável. Fizemos 4 tipos de teste.

### Teste 1 — Tratamento Defasado (C2 — Causalidade Reversa)

**O problema:** medimos D (receita de vendas) e Y (sobrepreço das compras) na **mesma temporada**.
Isso deixa a dúvida: e se o clube primeiro decidiu comprar caro e depois vendeu para pagar?
Nesse caso, a compra causaria a venda — e não o contrário.

**A solução:** usamos a receita de vendas do **ano anterior** (t-1) como tratamento alternativo.
A receita de t-1 é *predeterminada* em relação à compra de t — uma compra de 2024 não pode ter
causado as vendas de 2023.

**Resultado (SE clusterizado):** θ_defasado = **0,0051\*** (IC [0,0012; 0,0090]) — significativo.

**O que significa:** quando usamos a liquidez do ano anterior, o efeito persiste e é
significativo. A direção "venda → prêmio" fica mais plausível e **a causalidade reversa (C2)
enfraquece** — não eliminada.

> **Ressalva honesta:** na *mesma* subamostra, o contemporâneo é **não significativo** (θ=0,0055)
> mas o defasado é significativo. Isso é estranho se a história fosse puramente contemporânea — o
> lag provavelmente capta um **traço persistente de clube** ("clube que sempre vende", corr=0,44),
> parte do confundidor de porte. Logo o teste é **direcionalmente tranquilizador, não conclusivo**.

### Teste 2 — D Alternativo (Especificações do Tratamento)

**O problema:** medimos liquidez como `D = log(receita em euros)`. Mas e se o mecanismo for
diferente? E se não for o volume de dinheiro, mas outra coisa?

Testamos 3 definições (SE clusterizado):

| Tratamento | Interpretação | θ | Significativo? |
|------------|---------------|---|----------------|
| D₁ = log(receita €) — **principal** | Volume de dinheiro recebido | 0,0044 | **Não** |
| D₂ = log(n° de vendas) | Quantas vendas foram feitas | **0,049** | **Sim** |
| D₃ = flag de venda > €30M | Fez uma venda blockbuster? | **0,075** | **Sim** |

**Leitura inicial (sedutora, mas frágil):** D₁ (volume em euros) não prediz sobrepreço, mas D₂
(número de vendas) e D₃ (venda blockbuster) sim. Isso *sugeria* que o mecanismo é "**sinalizar**
que precisa comprar" (urgência de reposição / venda de ativo-chave), não o volume monetário.

> ⚠️ **Teste do artefato "clube ativo" derruba a leitura.** Clubes com janela movimentada
> compram **e** vendem muito. Ao adicionar ao W controles de intensidade de janela (`n_buys` + log
> do gasto total), **D₂ e D₃ perdem completamente a significância** (D₂: θ=0,015, p=0,40; D₃:
> θ=0,005, p=0,86). Ou seja, o sinal de D₂/D₃ era em grande parte **intensidade de atividade**, não
> um canal causal limpo de sinalização.

**Conclusão honesta:** o "mecanismo de sinalização" é **exploratório e NÃO robusto** — não deve
ser apresentado como achado confirmatório. Fica como hipótese a investigar com dados de melhor
granularidade (datas diárias, urgência observável).

### Teste 3 — Subgrupos por Tier de Liga

Dividimos as transferências em duas categorias:

- **Top-4:** Bundesliga, Premier League, La Liga, Serie A (3.450 obs) → θ = 0,0041 (n.s.)
- **Ligas menores:** Jupiler Pro League, Liga Portugal, Ligue-1 (1.696 obs) → θ = 0,0106 (n.s.)

**O que significa:** nenhum grupo tem efeito significativo na média das 8 temporadas — coerente
com o ATE geral nulo. Mas a **direção** está certa: ligas menores têm θ maior, o que é consistente
com a teoria (clubes de ligas menores são tipicamente vendedores e sofrem mais pressão de
reposição). A falta de significância é provavelmente falta de poder estatístico (n menor).

### Teste 4 — Placebos (já descrito na Etapa 2)

D embaralhado: θ = 0,0024 (n.s.) ✅  
D com ruído: θ = 0,0009 (n.s.) ✅

---

## 7. IVB — Índice de Vulnerabilidade de Barganha

### O que é

O IVB é um índice que pontua cada clube de 0 a 1 com base em quanto ele sofre o efeito do
prêmio do vendedor quando tem liquidez.

$$IVB_c = \frac{\theta_c - \min(\Theta)}{\max(\Theta) - \min(\Theta)}$$

- **IVB ≈ 1 ("presa fácil"):** clube que paga os maiores sobrepreços quando capitalizado
- **IVB ≈ 0 ("negociador disciplinado"):** clube que não deixa a liquidez afetar suas compras

O CATE (efeito causal individual estimado por clube) vira o IVB após normalização.

### Como foi calculado

Usamos o **R-learner** (Nie & Wager, 2021) — uma variante do DML que estima um efeito por
observação ao invés de um único ATE médio. O CATE de cada observação é então agregado por clube
(média das transferências do clube que tenham ≥ 3 observações para estabilidade).

**Limitação importante:** o desenho original previa `CausalForestDML` do pacote econml. O econml
não instala no nosso ambiente (Python 3.12 + NumPy 2.x têm incompatibilidade de build). O
R-learner é uma aproximação válida, mas os números **não são diretamente comparáveis** com o
que o econml daria. Por isso o IVB deve ser tratado como **ilustrativo**.

### O problema do outlier e da normalização min-max

O ranking atual tem Nîmes Olympique no topo (IVB ≈ 0,52, apenas 8 obs) — um clube da Ligue 1 com
poucas transferências na base. O problema: a normalização min-max divide pelo range (max − min). Se
um outlier tem CATE muito alto, o max explode, e todos os outros clubes ficam comprimidos
perto de zero. **O Nîmes domina o denominador e distorce o ranking.**

Por isso não usamos o IVB como ranking definitivo, mas como prova de conceito de ferramenta de
inteligência de mercado.

### O que seria se funcionasse corretamente

Com `CausalForestDML` e sem o problema do outlier, o IVB seria uma ferramenta real para:
- Um clube identificar, antes de negociar, quais rivais são mais vulneráveis à exploração quando
  estão capitalizados
- Um clube verificar se ele mesmo é uma "presa fácil" e ajustar sua estratégia de timing

---

## 8. Os Achados: 2 Confirmatórios + 1 Exploratório

> Disciplina inferencial: tratamos como **confirmatório** apenas o que sobrevive a SE clusterizado
> e (no caso sazonal) à correção de múltiplas comparações. O resto é **exploratório / gerador de
> hipótese** e está rotulado como tal.

### Achado 1 (confirmatório) — A Correlação Mente

OLS ingênuo: θ = **0,0142\*** (significativo)  
DML causal: θ = **0,0044** (não significativo, inclusive com SE clusterizado)

A correlação bruta entre "recebeu de vendas" e "pagou sobrepreço" é ~3× maior que o efeito causal
real — e enganosamente significativa. É o confundidor "clube rico" contaminando a correlação.
Sem o Double ML, teríamos publicado um resultado falso.

**Mensagem:** correlação não é causalidade. O pipeline causal de duas etapas é o que torna o
resultado defensável.

### Achado 2 (confirmatório) — O Prêmio é Condicional ao Regime

Na média de 8 temporadas, o efeito é nulo. Mas em **2022 (+0,064\*) e 2023 (+0,047\*)** — o boom
pós-COVID — o prêmio emerge de forma significativa, **sobrevivendo a Bonferroni e FDR**. Nos
outros 6 anos, não há efeito detectável.

**Mensagem:** o "efeito dominó" não é uma lei do mercado — é um fenômeno emergente em condições
específicas de escassez e urgência. O timing importa.

### Achado 3 (EXPLORATÓRIO, não robusto) — Sinalização vs. Volume

Na especificação bruta, o número de vendas (D₂, +0,049\*) e a venda blockbuster (D₃, +0,075\*)
prediziam sobrepreço, enquanto o volume em euros (D₁) não — o que *sugeria* um mecanismo de
"sinalização". **Porém**, ao controlar por intensidade de janela (`n_buys` + gasto total), **D₂ e
D₃ perdem toda a significância** (p=0,40 e p=0,86).

**Mensagem honesta:** o sinal de D₂/D₃ reflete em grande parte "clube com janela movimentada", não
um canal causal limpo de sinalização. **Este achado NÃO deve ser apresentado como confirmatório** —
é uma hipótese a investigar com dados de melhor granularidade.

---

## 9. O Que Não Fizemos e Por Quê

É importante ser honesto sobre o que prometemos e não entregamos, para não ser pego de surpresa
na banca.

### PSM (Propensity Score Matching)

**Prometemos:** o checkpoint mencionava "clubes gêmeos" via PSM para validar C6 (viés de seleção).

**Não fizemos porque:** o PSM exige que tratados e controles tenham distribuições de confundidores
sobreponíveis. Com 19 confundidores, o matching seria complexo e provavelmente reduziria muito
a amostra. Com o tempo disponível, priorizamos o DML (que controla confundidores de forma mais
flexível) e os testes de robustez.

**Como responder na banca:** "O PSM foi substituído pelo controle não-paramétrico via DML, que
trata os confundidores de forma mais flexível. Está declarado como trabalho futuro."

### FBref / StatsBomb (dados de performance)

**Prometemos:** integrar gols, xG, assistências, minutagem para melhorar o modelo hedônico.

**Não fizemos porque:** o entity matching entre Transfermarkt e FBref exige reconciliar IDs de
jogadores entre os dois sistemas — um processo demorado e sujeito a erros. Com o R² = 0,762 já
na faixa da literatura, a prioridade foi a robustez causal, não o refinamento do hedônico.

**Como responder na banca:** "O R² de 0,762 está na faixa esperada pela literatura (0,70–0,80).
A performance individual enriqueceria o hedônico mas está além do escopo atual."

### econml / CausalForestDML

**Prometemos:** usar `CausalForestDML` do econml para o IVB.

**Não fizemos porque:** incompatibilidade de build no ambiente (Python 3.12 + NumPy 2.x). O
R-learner foi usado como substituto para o CATE/IVB. O ATE e os testes de robustez usam DML
manual que é equivalente ao econml para o modelo parcialmente linear.

**Como responder na banca:** "O ATE e a robustez foram calculados com DML manual validado
(reproduz o econml dentro de erro de arredondamento). O IVB é ilustrativo — declaramos essa
limitação explicitamente."

### Análise do lado vendedor ("clubes predadores")

O IVB mede quem é vulnerável quando compra. A análise simétrica seria: quais clubes se
aproveitam melhor quando o outro lado está capitalizado e urgente? Isso não foi implementado
— mencionamos como trabalho futuro.

---

## 10. Glossário Rápido

| Termo | Definição simples |
|-------|-------------------|
| **Modelo hedônico** | Modelo que prevê o preço de um bem pelos seus atributos. Aqui: preço do jogador pelos seus atributos esportivos. |
| **Resíduo hedônico** | Diferença entre o preço pago e o preço previsto pelo modelo. Nosso Y = o sobrepreço. |
| **ATE** | Average Treatment Effect — efeito médio causal do tratamento (liquidez) sobre o resultado (sobrepreço). |
| **CATE** | Conditional Average Treatment Effect — efeito causal estimado individualmente para cada clube/observação. |
| **DML** | Double Machine Learning — método causal que usa dois modelos ML para eliminar confundidores não-lineares. |
| **Ortogonalização de Neyman** | A técnica dentro do DML de tirar o efeito de W de Y e de D antes de estimar θ. |
| **Cross-fitting** | Técnica de dividir os dados em K folds para evitar overfitting nos modelos de primeira etapa do DML. |
| **HC1** | Tipo de erro-padrão robusto à heterocedasticidade. Mais conservador que o SE clássico. |
| **Confundidor** | Variável que afeta tanto o tratamento quanto o resultado, criando correlação espúria. |
| **Causalidade reversa (C2)** | Quando o resultado causa o tratamento, não o contrário. Aqui: "comprou caro então vendeu" ao invés de "vendeu então comprou caro". |
| **R-learner** | Algoritmo de CATE que segue a lógica do DML mas estima efeitos individuais. Usado como substituto do CausalForestDML. |
| **IVB** | Índice de Vulnerabilidade de Barganha — score 0–1 do quanto cada clube sofre o prêmio quando capitalizado. |
| **PageRank** | Métrica de centralidade em grafo. Aqui: importância estrutural do clube na rede de transferências. |
| **Big sale flag** | Variável binária: 1 se o clube fez uma venda única acima de €30M na temporada. |
| **Split temporal** | Divisão de dados onde treino = passado e teste = futuro. Mais honesto que divisão aleatória para dados com tendência temporal. |
| **Winsorização** | Truncar os extremos da distribuição (aqui, percentis 1% e 99%) para reduzir o impacto de outliers. |
| **SHAP** | Técnica de interpretabilidade que calcula a contribuição de cada feature para cada predição individual. |
| **IC 95%** | Intervalo de Confiança de 95%. Se cruza zero, o efeito não é estatisticamente significativo a 5%. |

---

*Documento gerado em 18/06/2026. Dúvidas: releia as seções relevantes ou consulte os notebooks
em `notebooks/` e a documentação técnica em `knowledge/`.*

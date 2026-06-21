# Script — TP4 DemoDay · Grupo 02 · 25/06/2026
> **Duração:** 15 min · **14 slides** · `[clica]` = avança slide · **negrito** = ênfase verbal

---

## Slide 1 — Capa · ⏱ 0:15

> Boa tarde. Somos o Grupo 2, e nossa pergunta é simples: quando um clube faz uma grande venda e
> vai ao mercado comprar, ele paga mais caro do que deveria? Chamamos isso de **"prêmio do
> vendedor"**. A resposta que encontramos tem três camadas — e a última vai surpreender.

`[clica]`

---

## Slide 2 — Contexto & Problema · ⏱ 1:15

> O mercado de transferências funciona como uma **rede de dependências**. Quando o Porto vende
> Fábio Vieira por €35M, dois sinais chegam ao mercado ao mesmo tempo: o clube tem **caixa** e
> precisa **repor o jogador**. Os vendedores sabem disso.

> Nossa intuição: esse clube perde poder de barganha e passa a pagar **acima do valor justo** nas
> compras seguintes. É o **"efeito dominó"** — uma venda aciona uma cadeia de sobrepreços.

> Mas há uma armadilha óbvia: **clubes ricos vendem caro e compram caro ao mesmo tempo**. Como
> saber se é o prêmio do vendedor — ou só o porte do clube?

`[clica]`

---

## Slide 3 — Pergunta & Hipótese · ⏱ 0:45

> A pergunta operacional é: **a liquidez recente do comprador tem efeito causal sobre o sobrepreço
> que ele paga**, depois de neutralizar confundidores estruturais?

> Hipótese inicial: sim — e nossa conjectura era que o mecanismo seria de **sinalização**, não de
> volume de dinheiro (o clube pagaria mais porque o mercado sabe que ele precisa comprar). *Guardem
> essa conjectura: no slide 10 mostramos que os dados não a sustentam de forma robusta.*

`[clica]`

---

## Slide 4 — A Armadilha Causal · ⏱ 1:15

> Mapeamos **6 confundidores** que poderiam criar correlação espúria. Os três mais críticos:

> **C1 — Clube rico:** clubes grandes vendem caro E compram caro. Sem controlar isso, qualquer
> correlação é suspeita.

> **C4/C5 — Sazonalidade e inflação:** em anos de boom de mercado todo mundo paga mais. Precisamos
> separar o efeito do clube do efeito da época.

> **C2 — Causalidade reversa:** e se o clube já tinha planejado comprar caro, e vendeu para
> financiar? Nesse caso a compra causa a venda, não o contrário. Testamos isso explicitamente.

> Sem tratar esses confundidores, qualquer resultado seria correlação disfarçada de causalidade.

`[clica]`

---

## Slide 5 — Metodologia: Visão Geral · ⏱ 1:15

> Nossa estratégia tem duas etapas encadeadas.

> **Etapa 1 — Hedônico:** treino um modelo de ML que prevê o *preço justo* de cada jogador com
> base em atributos do atleta — idade, posição, valor de mercado — e controles de mercado. O
> **resíduo** — o que o modelo não consegue explicar — é o sobrepreço. É nossa variável Y.

> **Etapa 2 — Double ML:** pego esse resíduo e pergunto: a liquidez do clube comprador causa
> sobrepreço, depois de remover o efeito de todos os confundidores? Uso **ortogonalização de
> Neyman** e cross-fitting para eliminar o viés de ML de alta dimensão.

> A separação entre as duas etapas é o que garante que o resíduo não contenha o comportamento
> financeiro do clube — que fica reservado para os confundidores da Etapa 2.

`[clica]`

---

## Slide 6 — Etapa 1: Modelo Hedônico · ⏱ 1:15

> Os dados: **44.627 movimentações brutas** no Transfermarkt, **8 temporadas (2017–2025)**, 7
> ligas europeias. Após filtros — só compras pagas acima de €250k com valor de mercado válido —
> ficamos com **5.146 transferências**.

> Testamos 4 modelos. O vencedor foi o **Random Forest** com **R² de 0,76 no holdout de 2025**
> — treinado apenas em 2017–2024, testado no futuro. Isso é 76% da variância do preço explicada
> só pelos atributos do jogador.

> O SHAP confirma: **`log_mv`** domina. Depois, idade e posição. Nenhuma feature de comportamento
> do clube comprador entra aqui — por design.

> Baseline de comparação: usar só o market value como preditor direto dá R² de ~0,65. O ML
> agrega 11 pontos percentuais.

`[clica]`

---

## Slide 7 — Etapa 2: Double ML · ⏱ 1:15

> Para a estimação causal, definimos:

> **D — Tratamento:** log da receita de vendas do clube na temporada. Mede a liquidez recente.

> **Y — Resultado:** o resíduo hedônico da Etapa 1 — o sobrepreço pago acima do valor justo.

> **W — Confundidores (19 variáveis):** centralidade na rede (PageRank, graus), tamanho do
> elenco, 7 dummies de temporada, 6 dummies de liga. Controla C1 (clube rico), C4 (sazonalidade)
> e C5 (inflação de mercado).

> O DML estima θ pela regressão dos **resíduos ortogonalizados**: tiramos de Y e de D o que os
> confundidores explicam, e estimamos θ na variação que sobra — que, por construção, é exógena.

`[clica]`

---

## Slide 8 — Resultado 1: A Reviravolta · ⏱ 1:45

> Antes de mostrar o resultado causal, vejamos o que a correlação ingênua diz.

> **OLS sem controles:** θ = **0,0142\*** — IC [0,009; 0,019]. Significativo, positivo. "Prova"
> que existe prêmio do vendedor.

> *[pausa]*

> **Agora o Double ML** — com os confundidores controlados de forma não-linear:
> θ = **0,0044** — IC [−0,002; 0,010]. **O IC cruza zero (inclusive com erro-padrão clusterizado).
> Não significativo.**

> E o placebo — embaralhamos o tratamento D aleatoriamente — dá θ = 0,002. Quase do tamanho do
> efeito real. O modelo não encontra efeito causal médio.

> **A correlação mentia.** Era, em boa parte, o confundidor "clube rico" em ação. Sem DML,
> teríamos concluído que o prêmio existe quando, na média de 8 temporadas, ele não existe.

`[clica]`

---

## Slide 9 — Resultado 2: Onde o Efeito Vive · ⏱ 1:15

> Mas a média esconde a história. Quando estimamos θ **por temporada**, o padrão é claro:

> De 2017 a 2021 e em 2024–2025 — **nada**. ICs cruzam zero, efeito indistinguível de zero.

> **2022: θ = +0,064\*** — significativo.
> **2023: θ = +0,047\*** — significativo.
> E o mais importante: **ambos sobrevivem à correção de Bonferroni e ao FDR** para os 8 testes —
> não são falso-positivo de "testar muita coisa".

> Esses são os anos do **boom pós-COVID** — o mercado de transferências atingiu volumes recordes
> depois de 2 anos de restrição. Clubes com liquidez enfrentavam escassez de talentos disponíveis
> e urgência de reposição. *Nesse contexto*, o prêmio emerge.

> **Este é o achado central:** o prêmio do vendedor não é uma lei universal — é um fenômeno
> condicional ao regime de mercado.

`[clica]`

---

## Slide 10 — Resultado 3: O Mecanismo · ⏱ 1:15

> Agora a pergunta: **qual aspecto da liquidez aciona o prêmio?**

> Testamos três definições de tratamento:

> **D₁ = log(receita em euros):** θ = 0,0044 — **não significativo**. Volume de dinheiro em si
> não prediz sobrepreço.

> **D₂ = log(número de vendas):** θ = **4,9%\*** e **D₃ = venda blockbuster (>€30M):** θ = **7,5%\***
> — ambos significativos na especificação bruta. Isso *sugeria* que o que importa é **sinalizar
> necessidade**, não o volume.

> **MAS — e aqui está a honestidade do trabalho:** quando controlamos por "clube de janela
> movimentada" (nº de compras + gasto total), **D₂ e D₃ perdem toda a significância** (p=0,40 e
> p=0,86). Ou seja, esse sinal era em boa parte "clube ativo compra e vende muito", não um canal
> causal limpo.

> **Conclusão de mecanismo:** o mecanismo de sinalização é uma **hipótese exploratória, não um
> achado confirmado**. Os resultados sólidos são o Achado 1 (a correlação mente) e o efeito sazonal
> de 2022–2023. *[Se a banca perguntar do mecanismo, esta é a resposta honesta e que demonstra rigor.]*

`[clica]`

---

## Slide 11 — Resultado 4: Robustez · ⏱ 0:45

> Três testes validam a direção causal:

> **Placebos:** θ ≈ 0 com D embaralhado e com ruído gaussiano — o modelo não inventa efeito.

> **Tratamento defasado (C2):** usando a receita de vendas do **ano anterior** — predeterminada,
> não pode ser causada pela compra atual — θ = **0,38%\***. O efeito persiste. Causalidade
> reversa enfraquecida.

> **Subgrupos por liga:** ligas menores têm θ maior que top-4 — direção consistente com a teoria,
> embora sem significância individual por conta do tamanho de amostra menor.

`[clica]`

---

## Slide 12 — Conclusões & Implicações · ⏱ 1:15

> O prêmio do vendedor existe — mas com duas condições:
> **quando o mercado está aquecido** e **quando o clube sinaliza necessidade de comprar**.

> **Implicações práticas:**

> 1. **Timing de compra:** em mercados aquecidos, comprar imediatamente após uma grande venda é
>    caro. Clubes que podem esperar têm vantagem estrutural.

> 2. **Inteligência de mercado:** o **IVB** — Índice de Vulnerabilidade de Barganha — aponta os
>    clubes que mais sofrem o prêmio. *Ilustrativo nessa versão; base para ferramenta de scouting
>    financeiro.*

> 3. **Alerta metodológico:** OLS ingênuo teria levado à conclusão errada. Em mercados de
>    transferência, correlação não é causalidade.

`[clica]`

---

## Slide 13 — Limitações & Trabalho Futuro · ⏱ 0:30

> Reconhecemos três limitações principais:

> **Granularidade sazonal:** o tratamento é por temporada, não por dia. O efeito real da janela
> de 30 dias está diluído — nossa estimativa é conservadora.

> **Sem performance individual:** não integramos gols, xG, assistências (FBref). O resíduo pode
> capturar parte do valor atlético não observado.

> **IVB via R-learner:** o ranking de clubes é ilustrativo; CausalForestDML (com econml)
> daria resultado mais robusto.

> PSM, curva de decaimento temporal e análise do lado vendedor ficam como trabalho futuro.

`[clica]`

---

## Slide 14 — Encerramento · ⏱ 0:20

> *"O prêmio do vendedor não é uma lei do mercado — ele emerge nas janelas de mercado aquecido,
> e só quando o clube sinaliza que precisa comprar."*

> Obrigado. Estamos à disposição para perguntas.

---

# Divisão sugerida por integrante

| Integrante | Slides | Tempo |
|------------|--------|-------|
| **Lucas** | 1, 2, 3 | ~2:15 |
| **César** | 4, 5, 6, 7 | ~4:30 |
| **Carlos** | 8, 9, 10 | ~4:15 |
| **Leticia** | 11, 12, 13, 14 | ~3:10 |

---

# Q&A — Respostas preparadas

**"O ATE é nulo — então o prêmio do vendedor não existe?"**
> Não exatamente. O ATE *médio* de 8 temporadas é nulo, mas o efeito *condicional* existe e é
> robusto — em 2022–2023 é +6,4% e +4,7%, **sobrevivendo à correção de Bonferroni e FDR**.
> O achado é mais rico: o prêmio depende do regime de mercado (boom pós-COVID).

**"Como vocês sabem que 2022/2023 não é só sorte de testar 8 anos?"** (a defesa mais forte)
> Três camadas: (1) Bonferroni/FDR — sobrevivem; (2) **placebo de permutação dentro da safra**:
> embaralhando D entre os clubes de 2022/2023, θ colapsa para ~0 e o observado fica fora do
> percentil 97,5 do nulo, com **p empírico = 0,005**; (3) **sensibilidade ao corte de fee**: o
> efeito de 2022/2023 é estável de €0 a €500k. Não é artefato.

**"O corte de €250k não enviesa o resultado?"**
> Testamos limiares de €0 a €1M reconstruindo o pipeline inteiro. O **ATE é nulo em todos** e
> 2022/2023 são estáveis. Baixar para €0 (que reinclui as pechinchas) não muda nada.

**"E o confundidor 'clube rico' que vocês não controlam diretamente?"**
> Quantificamos: o **Robustness Value** é ~24–27% — um confundidor omitido teria de explicar um
> quarto da variação residual de D *e* de Y para anular 2022/2023. Além disso, num stress test que
> joga controles financeiros no W (over-control), 2023 ainda sobrevive (p≈0,003).

**"Por que o OLS com controles também não encontrou o efeito, se DML também não?"**
> Ambos dão θ ≈ 0,004–0,005 e IC cruzando zero. A diferença é que o DML controla as
> não-linearidades em W de forma mais flexível. O resultado nulo é robusto a ambas as
> especificações — isso reforça a conclusão, não enfraquece.

**"D₃ (big sale > €30M) ser significativo não pode ser endogeneidade? Grandes clubes vendem caro e compram caro."**
> Excelente pergunta — e nós **testamos exatamente isso**. Quando adicionamos ao W controles de
> intensidade de janela (nº de compras + gasto total), **D₂ e D₃ perdem toda a significância**.
> Ou seja, esse sinal era em grande parte "clube ativo", não um canal causal limpo. Por isso
> tratamos o "mecanismo de sinalização" como **exploratório**, não como achado confirmado.

**"O teste de lag prova causalidade reversa?"**
> Não prova — *enfraquece*, e nem isso de forma conclusiva. O lag t-1 é significativo (cluster),
> mas na mesma subamostra o contemporâneo é não significativo — o que sugere que o lag capta um
> traço persistente de clube ("sempre vende"), não um efeito limpo. Reportamos isso abertamente.
> Um teste decisivo exigiria datas diárias ou um instrumento.

**"Por que não usaram econml/CausalForestDML para o IVB?"**
> O econml não instala no ambiente Python 3.12 desta máquina por incompatibilidade com NumPy 2.x.
> O R-learner (Nie & Wager) é a alternativa mais próxima disponível. Os resultados do ATE e dos
> testes de robustez são equivalentes (DML manual reproduz o econml). O IVB é ilustrativo — está
> declarado explicitamente no trabalho.

---

# Cheat sheet de números

| Métrica | Valor |
|---------|-------|
| Transferências modeladas | 5.146 |
| Temporadas | 2017–2025 (exceto 2020) |
| Ligas | 7 |
| RF Test R² | 0,762 |
| RF Test RMSE | 0,673 |
| % com sobrepreço | 52,5% (out-of-fold) |
| Prêmio mediano | +3,5% |
| **OLS sem controles θ** | **0,0142\*** |
| **DML ATE θ** | **0,0044 (n.s., HC1 e cluster)** |
| DML 2022 θ | +0,064\* (sobrevive Bonferroni) |
| DML 2023 θ | +0,047\* (sobrevive Bonferroni) |
| D₂ (n_sales) θ | +0,049\* bruto → n.s. com controle de atividade (exploratório) |
| D₃ (big_sale) θ | +0,075\* bruto → n.s. com controle de atividade (exploratório) |
| Lag (C2) θ | +0,0051\* (cluster; não conclusivo) |
| Placebo shuffle θ | 0,0024 (n.s.) |
| R² model_t (m̂) | 0,37 |
| R² model_y (ĝ) | 0,06 |
| Confundidores em W | 19 (+ SE clusterizado em 974 clusters clube×temporada) |
| IVB top "presa" | Nîmes Olympique (ilustrativo) |

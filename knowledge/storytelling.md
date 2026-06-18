# Storytelling V1 — base para TP4 (DemoDay) e TP5 (artigo SBC)

> **Status:** rascunho V1 para evoluirmos juntos. Baseado nos resultados **reais de 8 temporadas**
> (ver [resumo.md](resumo.md) e [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md)).
> Entregas: **TP4** apresentação 15 min (18/06) · **TP5** artigo SBC (25/06).

---

## 1. A espinha narrativa (a história em um parágrafo)

O mercado de transferências é uma rede de dependências: quando um clube faz uma grande venda, fica
com caixa e urgência de reposição — e os rivais sabem disso. Será que ele passa a pagar um
**"prêmio do vendedor"** nas compras seguintes? Para responder com rigor (e não cair em correlação
espúria de "clube rico"), montamos um pipeline causal de duas etapas: um **modelo hedônico** que
estima o preço justo de cada jogador e isola o **sobrepreço**, e um **Double Machine Learning** que
mede o efeito causal da liquidez sobre esse sobrepreço, neutralizando confundidores. **A
descoberta:** na média de 8 temporadas o prêmio é **estatisticamente nulo** — mas ele **emerge,
significativo, exatamente nos anos de mercado aquecido (2022–2023, o boom pós-COVID)**. Ou seja, o
"efeito dominó" não é uma lei do mercado; é um fenômeno **condicional ao regime**. E como a liquidez
do **ano anterior** também prevê o sobrepreço, a explicação de "causalidade reversa" perde força.

### Frase-síntese (use em capa/encerramento)
> *"O prêmio do vendedor não é uma lei do mercado — ele emerge nas janelas de mercado aquecido.
> Na média, desaparece."*

---

## 2. Mensagens-chave (o que a audiência deve levar)

1. **Rigor causal > correlação.** Vender e comprar caro andam juntos — mas isso é, em boa parte,
   "clube rico". Nosso desenho separa o efeito *causal* da liquidez do simples porte do clube.
2. **O achado honesto é a condicionalidade temporal.** O prêmio existe **quando o mercado está
   quente** (2022–2023), não sempre. Liderar por isso (e não por um ATE médio frágil) é o que
   diferencia o trabalho.
3. **Validação séria.** Placebos limpos + teste de tratamento defasado (enfraquece causalidade
   reversa) dão credibilidade ao resultado — inclusive ao resultado *nulo*.
4. **Valor prático:** *timing* importa. Comprar logo após capitalizar, em mercado aquecido, é caro;
   o conceito de **IVB** (vulnerabilidade de barganha) aponta quem mais sofre — uma ferramenta de
   inteligência de mercado.

---

## 3. TP4 — Roteiro da apresentação (15 min, ~13 slides)

Cobre os 4 eixos exigidos: **Contexto · Metodologia · Resultados vs baseline · Conclusões/implicações**.
Tempo entre parênteses.

| # | Slide | Conteúdo / fala | t |
|---|-------|-----------------|---|
| 1 | **Capa** | Título, grupo, a frase-síntese como subtítulo | 0:20 |
| 2 | **Contexto & problema** | Mercado como rede de dependências; venda → caixa + urgência; "será que pago mais caro depois?" | 1:30 |
| 3 | **Pergunta & hipótese** | Pergunta operacional: liquidez recente → sobrepreço? Hipótese: é causal, não só "clube rico" | 1:00 |
| 4 | **A armadilha causal** | Os 6 confundidores (foco em C1 clube-rico, C4 sazonalidade, C2 reversa) — *por que correlação engana* | 1:30 |
| 5 | **Metodologia — visão geral** | Diagrama das 2 etapas (hedônico → DML). "Primeiro tiro o jogador da conta, depois isolo a liquidez" | 1:30 |
| 6 | **Etapa 1 — hedônico (+ baseline)** | Dados (44k→5,1k; 8 temporadas, 7 ligas). **Baseline = só market value**; ML (RF) → **Test R² 0,76**. Resíduo = sobrepreço | 1:30 |
| 7 | **Etapa 2 — Double ML** | D = liquidez (log receita de vendas), Y = resíduo, W = 19 confundidores. Intuição de ortogonalização | 1:30 |
| 8 | **Resultado 1 — a reviravolta** | **Baseline ingênuo (correlação/OLS) sugere prêmio; o DML mostra ATE ≈ 0 (n.s.)**. Placebos confirmam. *Aqui está o gancho dramático* | 2:00 |
| 9 | **Resultado 2 — onde o efeito vive** | θ por temporada: **2022 (+5,2%\*) e 2023 (+3,9%\*)** significativos; resto ≈ 0. Gráfico forest. *O achado central* | 1:30 |
| 10 | **Resultado 3 — robustez & heterogeneidade** | Teste de lag (C2 enfraquecida); IVB como ferramenta (ilustrativo, com ressalva) | 1:00 |
| 11 | **Conclusões & implicações** | Não é lei universal → é condicional ao regime. Valor: *timing* de compra; inteligência de mercado | 1:30 |
| 12 | **Limitações & futuro** | Granularidade sazonal; sem performance (FBref); IVB via R-learner; PSM como trabalho futuro | 0:45 |
| 13 | **Encerramento** | Recapitula a frase-síntese; obrigado/perguntas | 0:25 |

**Dicas de DemoDay:** abrir com a intuição (o "dominó") antes do método; segurar o suspense do ATE
nulo para o slide 8; no slide 9 entregar o "plot twist" (o efeito existe, mas só em 2022–2023);
fechar no valor prático. Cada integrante domina 2–3 slides.

---

## 4. TP5 — Estrutura do artigo SBC (mapa de seções)

| Seção SBC | O que entra | Fontes no knowledge |
|-----------|-------------|---------------------|
| **1. Introdução** | Contexto do mercado-rede; problema (prêmio do vendedor); pergunta de pesquisa; contribuições (pipeline causal + achado condicional); organização do artigo | [resumo.md](resumo.md) |
| **2. Trabalhos Relacionados** | Redes de transferência (Li et al., Palazzo, Dieles); valuation Transfermarkt (Peeters); inferência causal (Chernozhukov DML; Nie & Wager R-learner; Angrist & Pischke). *Diferencial: foco no preço/contexto, não na estrutura da rede* | proposta (deliverables) |
| **3. Metodologia** | Dados (Transfermarkt, 8 temporadas, 7 ligas, funil 44k→5,1k); feature engineering (clube + rede); Etapa 1 hedônica (target, 19 features, split temporal, 4 modelos); Etapa 2 DML (D, Y, W, ortogonalização, cross-fitting, HC1); IVB | [dados_e_features.md](dados_e_features.md), [hedonic_ml.md](hedonic_ml.md), [double_ml.md](double_ml.md) |
| **4. Resultados e Discussão** | **Baseline vs modelos** (MV puro vs ML, R² 0,76; correlação ingênua vs DML); **ATE nulo** + placebos + 1ª etapa; **θ por temporada (2022–2023)**; **lag/C2**; **IVB** (ilustrativo). Discussão: por que a média some e o efeito é condicional | [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md), [double_ml.md](double_ml.md) |
| **5. Aplicações (implicações práticas)** | Inteligência de mercado: *timing* de compra em mercado aquecido; IVB para sinalizar vulnerabilidade; alerta contra ler correlação como causa em scouting financeiro | resumo §6 |
| **6. Conclusão e Trabalhos Futuros** | Síntese (prêmio condicional ao regime); contribuições; futuro: performance (FBref), datas diárias (curva de decaimento), PSM, IVB com CausalForestDML, lado vendedor ("predadores") | [pendencias.md](pendencias.md) |
| **7. Referências** | SBC/ABNT; já há 8 refs na proposta — completar com Chernozhukov (2018) e Nie & Wager | proposta |

**Tom do artigo:** assumir o resultado nulo médio como **contribuição** (refuta a intuição ingênua
do dominó universal) e destacar a **heterogeneidade temporal** + a validação como rigor científico.

---

## 5. Decisões em aberto (vamos resolver juntos)

1. **Framing do título/abstract:** "prêmio condicional ao mercado aquecido" (recomendado) vs. manter
   o enquadramento "efeito dominó" com a ressalva. Sugiro o primeiro, citando o segundo como motivação.
2. **IVB nos slides:** incluir como *ferramenta ilustrativa* (com a ressalva do R-learner/outlier)
   ou deixar só no artigo? Sugiro 1 slide leve + detalhe no artigo.
3. **Baseline a destacar:** confirmar que usamos (a) MV puro como baseline do hedônico e (b)
   correlação/OLS ingênua como baseline causal. Vale rodar o OLS ingênuo p/ ter o número do contraste.
4. **2021 (COVID) e 2020 (ausente):** manter com nota metodológica (recomendado) ou excluir 2021
   numa análise de robustez.
5. **Reprocessar IVB com econml** (Colab) antes da entrega? Decide se o IVB entra como resultado
   "firme" ou "ilustrativo".

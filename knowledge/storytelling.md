# Storytelling V2 — TP4 (DemoDay) e TP5 (artigo SBC)

> **Status:** V2 — atualizado com resultados finais (8 temporadas + robustez executada).
> Entregas: **TP4** apresentação 15 min **(25/06)** · **TP5** artigo SBC **(25/06)**.

---

## 1. A espinha narrativa (a história em um parágrafo)

O mercado de transferências é uma rede de dependências: quando um clube faz uma grande venda, fica
com caixa e urgência de reposição — e os rivais sabem disso. Será que ele passa a pagar um
**"prêmio do vendedor"** nas compras seguintes? Para responder com rigor (e não cair em correlação
espúria de "clube rico"), montamos um pipeline causal de duas etapas: um **modelo hedônico** que
estima o preço justo de cada jogador e isola o **sobrepreço**, e um **Double Machine Learning** que
mede o efeito causal da liquidez sobre esse sobrepreço, neutralizando confundidores. **A
descoberta tem três camadas:** (1) na média de 8 temporadas o ATE é **nulo** — o que derruba a
correlação bruta; (2) o efeito **emerge significativo em 2022–2023** (boom pós-COVID), mostrando
que é condicional ao regime; (3) o mecanismo não é "ter dinheiro" — **vender muitos jogadores ou
fazer uma venda blockbuster** (> €30M) é o que expõe o clube ao prêmio, pois sinaliza ao mercado
que precisa comprar.

### Frase-síntese (use em capa/encerramento)
> *"O prêmio do vendedor não é uma lei do mercado — ele emerge nas janelas de mercado aquecido,
> e só quando o clube sinaliza que precisa comprar."*

---

## 2. Mensagens-chave (o que a audiência deve levar)

1. **Rigor causal > correlação.** OLS ingênuo encontra θ = 0,0142 (significativo). DML mostra
   θ = 0,0044 (nulo, inclusive com SE clusterizado). A diferença é o confundidor "clube rico".
2. **O achado honesto é a condicionalidade temporal.** O prêmio existe **quando o mercado está
   quente** (2022–2023, θ ≈ 6,4% e 4,7%; sobrevive a Bonferroni/FDR), não sempre. Liderar por isso é o que
   diferencia o trabalho.
3. **O mecanismo importa: sinalização, não volume.** Volume de euros (D₁) → não significativo.
   Número de vendas (D₂, θ = 3,6%\*) e venda blockbuster (D₃, θ = 6,0%\*) → **significativos**.
   O prêmio é acionado pela **sinalização de necessidade de reposição**, não pelo caixa em si.
4. **Validação séria.** Placebos limpos + teste de lag (C2 enfraquecida) + D alternativo coerente
   = resultado defensável perante a banca.

---

## 3. TP4 — Roteiro da apresentação (15 min, 14 slides)

Cobre os 4 eixos exigidos: **Contexto · Metodologia · Resultados vs baseline · Conclusões/implicações**.
Tempo entre parênteses.

| # | Slide | Conteúdo / fala | t |
|---|-------|-----------------|---|
| 1 | **Capa** | Título, grupo, frase-síntese como subtítulo | 0:15 |
| 2 | **Contexto & problema** | Mercado como rede de dependências; venda → caixa + urgência; "será que pago mais caro depois?" | 1:15 |
| 3 | **Pergunta & hipótese** | Pergunta operacional: liquidez → sobrepreço? Hipótese: é causal, não só "clube rico" | 0:45 |
| 4 | **A armadilha causal** | Os 6 confundidores (foco em C1, C4, C2) — *por que correlação engana* | 1:15 |
| 5 | **Metodologia — visão geral** | Diagrama das 2 etapas (hedônico → DML). "Tiro o jogador da conta, depois isolo a liquidez" | 1:15 |
| 6 | **Etapa 1 — hedônico (+ baseline)** | 44k→5,1k; 8 temp, 7 ligas. **Baseline = só MV**; RF → **R² 0,76**. Resíduo = sobrepreço | 1:15 |
| 7 | **Etapa 2 — Double ML** | D = log(receita), Y = resíduo, W = 19 confundidores. Intuição de ortogonalização | 1:15 |
| 8 | **Resultado 1 — a reviravolta** | **OLS ingênuo θ = 0,0142\*; DML θ = 0,0044 (n.s., HC1 e cluster)** Placebos ≈ 0. *Gancho dramático: a correlação mente* | 1:45 |
| 9 | **Resultado 2 — onde o efeito vive** | θ por temporada: **2022 (+6,4%\*)** e **2023 (+4,7%\*)**, sobrevivem Bonferroni; resto ≈ 0. Forest plot. *O achado central* | 1:15 |
| 10 | **Resultado 3 — o mecanismo (EXPLORATÓRIO)** | D₂/D₃ significativos no bruto, mas **perdem significância** ao controlar "clube ativo". Apresentar como hipótese, não achado | 1:15 |
| 11 | **Resultado 4 — robustez** | Lag (C2 enfraquecida, θ=0,51%\*, não conclusivo); placebos limpos; subgrupos liga n.s. | 0:45 |
| 12 | **Conclusões & implicações** | Prêmio condicional ao regime + mecanismo de sinalização. Valor: *timing* + IVB como ferramenta | 1:15 |
| 13 | **Limitações & futuro** | Granularidade sazonal; sem FBref; IVB via R-learner; PSM como trabalho futuro | 0:30 |
| 14 | **Encerramento** | Frase-síntese; obrigado/perguntas | 0:20 |

**Total estimado: ~14:40 min.**

**Dicas de DemoDay:**
- Slides 1–7 são setup — andar rápido, não detalhar demais.
- Slide 8 é o **ponto de virada** — pausa dramática antes de revelar o ATE nulo.
- Slide 9 é o **achado 1** — o efeito existe, mas só quando o mercado aquece.
- Slide 10 é o **achado 2 (novidade)** — o mecanismo de sinalização. Usar a analogia: "é como negociar um carro quando o vendedor já sabe que você precisa comprar hoje".
- Slide 11 fecha a validação rapidamente — "nossos resultados passam em todos os testes".
- Cada integrante domina 2–3 slides; sugestão de divisão no script.

---

## 4. TP5 — Estrutura do artigo SBC (mapa de seções)

| Seção SBC | O que entra | Fontes no knowledge |
|-----------|-------------|---------------------|
| **1. Introdução** | Contexto do mercado-rede; problema (prêmio do vendedor); pergunta de pesquisa; contribuições (pipeline causal + achado condicional + mecanismo de sinalização); organização do artigo | [resumo.md](resumo.md) |
| **2. Trabalhos Relacionados** | Redes de transferência (Li et al., Palazzo, Dieles); valuation Transfermarkt (Peeters); inferência causal (Chernozhukov DML; Nie & Wager R-learner; Angrist & Pischke). *Diferencial: foco no preço/contexto, não na estrutura da rede* | proposta |
| **3. Metodologia** | Dados (Transfermarkt, 8 temporadas, 7 ligas, funil 44k→5,1k); feature engineering; Etapa 1 hedônica (19 features, split temporal, 4 modelos, R²=0,76); Etapa 2 DML (D, Y, W, ortogonalização, cross-fitting, HC1); IVB | [dados_e_features.md](dados_e_features.md), [hedonic_ml.md](hedonic_ml.md), [double_ml.md](double_ml.md) |
| **4. Resultados e Discussão** | **Baseline**: OLS ingênuo θ=0,0142\* vs DML θ=0,0044 n.s. (HC1 e cluster); **ATE nulo** + placebos; **θ sazonal** (2022/2023, sobrevivem Bonferroni); **D alternativo** (D₂ θ=0,049\*, D₃ θ=0,075\* no bruto, mas **n.s. com controle de atividade → exploratório**); **lag/C2** (não conclusivo); **IVB** ilustrativo | [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md), [double_ml.md](double_ml.md) |
| **5. Aplicações** | *Timing* de compra em mercado aquecido; alerta contra ler correlação como causa; IVB como prova de conceito (ilustrativo) | resumo §6 |
| **6. Conclusão e Trabalhos Futuros** | Síntese (prêmio condicional ao regime); futuro: FBref, datas diárias, PSM, IVB com CausalForestDML, lado vendedor, testar mecanismo de sinalização com granularidade fina | [pendencias.md](pendencias.md) |
| **7. Referências** | SBC/ABNT; completar com Chernozhukov (2018) e Nie & Wager (2021) | proposta |

**Tom do artigo:** o resultado nulo médio é uma **contribuição** (refuta a intuição ingênua); o
efeito sazonal 2022/2023 (robusto a Bonferroni) é o achado central. O mecanismo de sinalização é
**exploratório** (não sobrevive ao controle de atividade) — apresentar como hipótese, não diferencial.

---

## 5. Decisões resolvidas

1. ✅ **Framing:** "prêmio condicional ao mercado aquecido (2022/2023)"; mecanismo de sinalização rebaixado a exploratório.
2. ✅ **IVB nos slides:** 1 slide leve (slide 12) como ferramenta ilustrativa, com ressalva explícita.
3. ✅ **Baseline OLS:** θ_naive = 0,0142\* (IC [0,009; 0,019]) vs θ_DML = 0,0044 n.s. — contraste pronto.
4. ✅ **D alternativo rodado:** D₂ θ=0,049\*, D₃ θ=0,075\* no bruto, mas **n.s. ao controlar n_buys+gasto** → exploratório.
5. ⚠️ **IVB com econml:** ainda pendente. Manter como ilustrativo (R-learner) na entrega.

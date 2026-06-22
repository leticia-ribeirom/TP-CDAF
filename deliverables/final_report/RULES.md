# RULES.md — Regras de escrita do Relatório Final (TP5, artigo SBC)

Documento normativo para escrever o relatório final. Derivado de **três papers-modelo** em
`deliverables/examples/` (todos do mesmo autor/grupo), do **enunciado** (`deliverables/Enunciado
projeto da disciplina.pdf`) e das decisões do grupo. **Ler antes de escrever qualquer seção.**

---

## 1. Objetivo e formato

- **Entregável:** artigo científico no **modelo SBC** (Sociedade Brasileira de Computação),
  similar ao FAME'26. Peso 15 pts. Entrega **25/06/2026**.
- **Idioma:** português (pt-br), técnico-formal.
- **Avaliação foca em:** profundidade da análise, rigor científico, clareza da escrita e aderência
  ao formato SBC.

### Decisão de template
Dos exemplos, `_UFMG__PPML___Trabalho_Final` segue o **formato SBC** (uma coluna, seções numeradas
`1.`, `2.`…, citações `[Autor et al. ano]`). Os dois POCs usam IEEE (duas colunas, `[N]`). **Como o
enunciado exige SBC, seguimos o estilo do PPML.**

- `\documentclass[12pt]{article}` + `\usepackage{sbc-template}` (template oficial SBC).
- `\usepackage[brazil]{babel}`, `\usepackage[utf8]{inputenc}`, `graphicx`, `amsmath`, `booktabs`,
  `url`.
- `\bibliographystyle{sbc}` (arquivo `sbc.bst`) + `\bibliography{references}`.
- Citações **textuais** com `\cite{}` no estilo SBC: *autor-data* → `[Chernozhukov et al. 2018]`.

---

## 2. Estrutura de arquivos (um `.tex` por seção)

```
deliverables/final_report/
├── RULES.md                  (este arquivo)
├── main.tex                  (preâmbulo SBC + metadados + \input das seções na ORDEM FINAL)
├── references.bib            (bibliografia — PRIMEIRO a ser escrito)
├── sections/
│   ├── 00_resumo_abstract.tex   (resumo PT + abstract EN)
│   ├── 01_introducao.tex        (escrita POR ÚLTIMO)
│   ├── 02_trabalhos_relacionados.tex
│   ├── 03_metodologia.tex
│   ├── 04_resultados_discussao.tex
│   ├── 05_aplicacoes.tex
│   └── 06_conclusao_futuro.tex
└── imgs/                     (figuras — gerenciadas no Overleaf pelo grupo; ver §8)

> **Figuras:** o grupo monta o LaTeX no Overleaf e cuida do upload das imagens. Ao inserir um
> gráfico no `.tex`, referenciar sempre o caminho **`imgs/<nome>.png`** (`\includegraphics`).
```

`main.tex` faz `\input{sections/01_introducao}` etc., **na ordem final do artigo** (Introdução
primeiro no PDF), independentemente da ordem em que escrevemos.

---

## 3. Ordem de ESCRITA (regras do grupo)

1. **`references.bib` primeiro** — fixar a bibliografia antes de redigir.
2. **`02_trabalhos_relacionados.tex` em seguida** — ancora a fundamentação.
3. Depois: Metodologia → Resultados e Discussão → Aplicações → Conclusão.
4. **Introdução por ÚLTIMO** (`01_introducao.tex`) — escrita quando todo o resto já existe, para
   prometer exatamente o que o artigo entrega.
5. Resumo/Abstract por último, junto da introdução.

> Ordem no **PDF final** ≠ ordem de escrita. No PDF: Resumo → Introdução → Trabalhos Relacionados →
> Metodologia → Resultados → Aplicações → Conclusão → Referências.

---

## 4. Seções obrigatórias (enunciado TP5) e o que entra em cada

| Seção | Conteúdo exigido | Fonte no repo |
|-------|------------------|---------------|
| **Resumo + Abstract** | 1 parágrafo cada; problema, método, achado principal, número-chave | `knowledge/resumo.md` |
| **1. Introdução** | contexto, definição do problema, **pergunta de pesquisa**, contribuições, **organização do artigo** | `knowledge/resumo.md`, `storytelling.md` |
| **2. Trabalhos Relacionados** | revisão; como cada trabalho **se assemelha / difere / fundamenta** o nosso | `knowledge/storytelling.md §4`, papers (§7) |
| **3. Metodologia** | dados (origem/volume), pré-processamento, features, pipeline 2 etapas, modelos, DML (W, ortogonalização, cross-fitting), métricas | `dados_e_features.md`, `hedonic_ml.md`, `double_ml.md` |
| **4. Resultados e Discussão** | quantitativo + qualitativo, **comparação com baseline**, interpretação, comparação com estado da arte | `double_ml.md`, `reprocessamento_8temporadas.md`, `resumo.md` |
| **5. Aplicações (Implicações Práticas)** | uso no mundo real (scouting, *timing*, inteligência de mercado, IVB) | `storytelling.md §2`, `resumo.md §6` |
| **6. Conclusão e Trabalhos Futuros** | síntese, contribuições, limitações, futuro (PSM, FBref, decaimento, lado vendedor) | `pendencias.md` |
| **7. Referências** | lista formatada SBC | `references.bib` |

---

## 5. Estilo de escrita (extraído dos exemplos)

**Voz e tempo.** Impessoal/1ª pessoa do plural acadêmica: *"Este trabalho investiga…", "propomos…",
"Os resultados mostraram…", "Adotamos…"*. Evitar "eu", "a gente", coloquialismos.

**Densidade técnica.** Alta, mas didática. Cada conceito-chave é **definido na primeira aparição**
(ex.: *"O modelo hedônico decompõe o preço de um bem na soma do valor de seus atributos."*). Não
assume que o leitor conhece DML/IVB.

**Termos estrangeiros em itálico:** `\textit{}` para *baseline*, *Double Machine Learning*,
*overfitting*, *holdout*, *cross-fitting*, *Random Forest*, *placebo*, etc. Na 1ª vez, sigla entre
parênteses: *"Double Machine Learning (DML)"*.

**Subseções inline em negrito.** Padrão SBC do PPML: parágrafos temáticos começam com rótulo em
negrito + ponto. Ex.: **`\textbf{Ortogonalização de Neyman.}`** seguido do texto.

**Contribuições enumeradas** na introdução: *"Nossas principais contribuições são: (i) …; (ii) …;
(iii) …"*.

**Posicionamento vs. literatura.** Usar a fórmula dos exemplos: *"Diferentemente de [X], que
[abordagem deles], nossa abordagem [diferença]."*

**Organização do artigo** ao fim da introdução: *"A Seção 2 apresenta… A Seção 3 detalha… A Seção 4
discute…"*.

**Parágrafos** com tópico-frase clara; transições explícitas (*"Contudo", "Além disso", "Por fim"*).

---

## 6. Convenções de elementos

**Tabelas.** Título **acima**, centralizado: `\caption{Tabela N. Título.}` (o template SBC numera).
Usar `booktabs` (`\toprule/\midrule/\bottomrule`), sem linhas verticais. Destacar a linha do
modelo vencedor (negrito). Toda tabela deve ser **referenciada no texto** ("a Tabela 1 apresenta…").

**Figuras.** Legenda **abaixo**, descritiva e autossuficiente: `\caption{Figura N. <descrição do
que mostrar + leitura>.}`. Reutilizar gráficos já existentes (§8). Toda figura referenciada no texto.

**Equações.** Numeradas com `equation`; variáveis em itálico matemático. Ex.:
`$Y_{i} = \theta_0 D_i + g(W_i) + U_i$`. Definir cada símbolo logo após.

**Números (pt-br).** Vírgula decimal: **0,762** (não 0.762). Percentuais com `\%`. Intervalos de
confiança: `IC 95\% [−0,0016; 0,0104]`. Significância: `\theta = 0{,}064` com `*` ou `p = 0{,}0006`.

**Citações.** `\cite{chernozhukov2018}` → `[Chernozhukov et al. 2018]`. Toda afirmação de literatura
ou método externo **deve** ter citação.

---

## 7. Bibliografia — entradas mínimas para `references.bib`

**Já citados (proposta TP1):** Wand 2022; Li, Zhou & Stanley; Palazzo et al.; Dieles, Mattsson &
Takes; Peeters; Angrist & Pischke; Imbens & Rubin; Cunningham.

**Métodos usados (obrigatórios):** Chernozhukov et al. 2018 (DML); Nie & Wager 2021 (R-learner);
Cinelli & Hazlett 2020 (Robustness Value); Benjamini & Hochberg 1995 (FDR); Breiman 2001 (Random
Forest); Lundberg & Lee 2017 (SHAP).

**A baixar (fecham o estado da arte do hedônico):** ⭐ McHale & Holmes 2023 (ML para transfer fees,
EJOR); ⭐ Müller, Simons & Weinmann 2017 (market value data-driven, EJOR); Herm, Callsen-Bracker &
Kreis 2014 (crowd valuation Transfermarkt).

> Regra: **não citar o que não vamos usar no texto**, e **não afirmar nada da literatura sem entrada
> no `.bib`**.

---

## 8. Figuras disponíveis no repositório (regra: reusar, não recriar)

As figuras estão **embutidas nos notebooks** — exportar para `imgs/` (script de export ou salvar do
notebook). Candidatas por seção:

| Figura | Origem | Seção sugerida |
|--------|--------|----------------|
| Distribuição do prêmio (resíduo) | `etapa1_hedonic_model.ipynb` | Metodologia/Resultados |
| SHAP — importância de features | `etapa1_hedonic_model.ipynb` | Resultados (Etapa 1) |
| **Baselines: OLS naive → OLS+W → DML** | `build_final_slides.py` / `etapa2` | Resultados (a reviravolta) ⭐ |
| **θ por temporada (forest plot 2017–2025)** | `etapa2_double_ml.ipynb` / `build_final_slides.py` | Resultados (achado central) ⭐ |
| Resíduos ortogonalizados | `etapa2_double_ml.ipynb` | Metodologia (DML) |
| Distribuição dos CATEs | `etapa2_double_ml.ipynb` | Resultados (heterogeneidade) |
| Grafo da rede / EDA de mercado | `exploratory_analysis.ipynb` (15 figs) | Introdução/Metodologia |

⭐ = as duas figuras-âncora do artigo. **Não inventar gráficos novos** sem necessidade; se faltar,
gerar a partir dos CSVs em `output/`.

---

## 9. Regras de conteúdo e rigor científico (críticas para este projeto)

1. **A manchete é honesta:** o ATE médio é **não significativo** (0,0044, IC cruza zero com HC1 e
   cluster). **Nunca** escrever "encontramos um prêmio do vendedor causal" como conclusão geral. O
   achado é a **heterogeneidade temporal**: efeito significativo só em 2022–2023 (boom pós-COVID),
   sobrevivendo a Bonferroni/FDR.
2. **Comparação com baseline é obrigatória** (enunciado): baseline preditivo (MV puro, R² 0,688) vs.
   ML (RF 0,762); baseline causal (OLS naive 0,0142\*) vs. DML (0,0044 n.s.).
3. **Reportar limitações abertamente** (como os exemplos fazem): granularidade sazonal; C1
   controlado por proxies (W sem volume financeiro direto); C3 não tratado; CATE/IVB via R-learner;
   resíduo sem performance individual; IVB ilustrativo (dominado por outlier).
4. **Todo número vem dos notebooks executados** — conferir contra `knowledge/` (já auditado e
   consistente). Não arredondar de forma que mude a interpretação.
5. **D₂/D₃ (sinalização) = exploratório**, não achado confirmado (perde significância ao controlar
   atividade do clube). Escrever exatamente assim.

---

## 10. Checklist por seção (antes de dar como pronta)

- [ ] Cobre o item correspondente do enunciado (§4).
- [ ] Todo número confere com os notebooks / `knowledge/`.
- [ ] Toda tabela e figura é referenciada no texto e tem legenda autossuficiente.
- [ ] Toda afirmação de literatura tem `\cite{}` com entrada no `.bib`.
- [ ] Termos estrangeiros em itálico; siglas definidas na 1ª aparição.
- [ ] Vírgula decimal; ICs e significância no padrão.
- [ ] Sem promessa que o artigo não cumpre (coerência com a manchete honesta).
- [ ] Compila no `main.tex` sem erro de referência cruzada.

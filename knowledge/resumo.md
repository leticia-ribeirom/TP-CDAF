# Resumo — Visão Geral do Projeto

> ⚠️ **Atualizado para 8 temporadas (2017–2025, sem 2020).** Os números abaixo refletem o
> reprocessamento. A descoberta principal **mudou** (o ATE deixou de ser significativo) — veja o
> changelog completo em [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md).

## 1. O tema

**Efeito Dominó no Mercado de Transferências: o "Prêmio do Vendedor" em Compras Subsequentes.**
Grupo 02 · Ciência de Dados Aplicada ao Futebol · UFMG.

O mercado de transferências funciona como uma rede de dependências: quando um clube faz uma
venda grande, fica com **caixa em mãos** e **urgência de reposição** — e os rivais sabem disso.
A intuição do projeto é que esse clube acaba pagando mais caro nas compras seguintes: o
**"prêmio do vendedor"**.

## 2. Pergunta de pesquisa

**Pergunta original (proposta):** dada uma venda do Time A em uma janela, qual a probabilidade e
a intensidade do sobrepreço que ele paga nas compras subsequentes da mesma janela?

**Pergunta operacional (como foi de fato modelada):** a **liquidez recente do comprador** (receita
de vendas) tem **efeito causal** sobre o **sobrepreço** que ele paga em novas contratações,
depois de neutralizar confundidores estruturais?

> ⚠️ A pergunta migrou de "probabilidade" para "intensidade do efeito causal + heterogeneidade".
> A granularidade diária (janela de 30 dias) prometida na proposta **não existe na base**; o
> tratamento foi operacionalizado por **temporada**. Ver [pendencias.md](pendencias.md).

## 3. A estratégia em duas etapas

O problema foi quebrado em duas etapas encadeadas:

```
            ETAPA 1 (Hedônico/ML)                 ETAPA 2 (Causal/DML)
   atributos do jogador  ──►  preço justo      receita de vendas (D)  ──►  efeito causal θ
        ln(fee)               estimado P̂                                    sobre o resíduo
                                  │                                              │
                                  ▼                                              ▼
                       resíduo = preço − preço justo  ───────────────►  prêmio de reinvestimento (Y)
                       ("prêmio de reinvestimento")
```

- **Etapa 1 — Modelo Hedônico (ML supervisionado).** Prevê `ln(fee)` a partir de atributos do
  jogador e controles de mercado. O **resíduo** (preço pago − preço justo estimado) é o que
  sobra depois de remover o valor intrínseco do atleta → é o nosso **prêmio**.
  Detalhes em [hedonic_ml.md](hedonic_ml.md).

- **Etapa 2 — Double Machine Learning.** Trata a receita de vendas do comprador como
  **tratamento (D)** e o resíduo hedônico como **resultado (Y)**, neutralizando confundidores
  estruturais (W) com ML duplo. Estima o efeito médio (ATE), a heterogeneidade (CATE) e o índice
  **IVB**. Detalhes em [double_ml.md](double_ml.md).

## 4. Dados (resumo)

- Fonte: dump do **Transfermarkt** (padrão `dcaribou/transfermarkt-datasets`).
- **44.627** movimentações brutas → após filtros (compras pagas, fee ≥ €250k, MV válido) e limpeza:
  **5.146** transferências usadas na modelagem.
- **8 temporadas** (2017–2025, sem 2020) · **7 ligas** europeias.
- Detalhes em [dados_e_features.md](dados_e_features.md).

## 5. Números-chave (resultados reais)

### Etapa 1 — Hedônico
| Métrica | Valor |
|---------|-------|
| Melhor modelo | **Random Forest** |
| Test R² (holdout 2025) | **0,762** |
| Test RMSE | 0,673 |
| Split | treino 2017–24 (4.396) · teste 2025 (750) |
| Feature dominante (SHAP) | `log_mv` (valor de mercado), depois `age`, `age_sq`, liga |
| Transferências com prêmio positivo | **51,7%** |
| Prêmio mediano (resíduo) | **+0,019** (≈ +1,9%) |

### Etapa 2 — Causal (Double ML)
> Calculada com **DML manual** (econml não instala no ambiente atual). CATE/IVB via **R-learner**
> (aproximação do CausalForestDML). Detalhes e ressalvas em
> [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md).

| Métrica | Valor |
|---------|-------|
| **ATE (θ)** | **0,0031** · IC 95% **[−0,0022; 0,0083]** → **NÃO significativo** |
| Placebo (D embaralhado) | θ = 0,0023 (≈ do tamanho do ATE → reforça nulidade) |
| Placebo (ruído gaussiano) | θ = 0,0006 · IC cruza zero |
| 1ª etapa: R² model_t (m̂) | 0,37 |
| 1ª etapa: R² model_y (ĝ) | 0,06 |
| **θ por temporada** | só **2022 (+0,052\*)** e **2023 (+0,039\*)** são significativos |
| **Teste de lag (C2)** | θ defasado = **0,0038\*** [0,0003; 0,0073] → enfraquece causalidade reversa |

### IVB — Índice de Vulnerabilidade de Barganha (190 clubes) — *ilustrativo (R-learner)*
- **"Presas fáceis" (IVB alto):** Nîmes Olympique, KV Mechelen, Sunderland.
- **"Negociadores disciplinados" (IVB baixo):** SD Eibar, Hellas Verona, Arminia Bielefeld.
- ⚠️ Ranking mudou de método e composição vs. a versão de 3 temporadas — tratar como ilustrativo
  até reprocessar com `CausalForestDML` (econml).

## 6. A descoberta de fato (e a tensão narrativa)

Com 8 temporadas, o efeito causal **médio** virou **estatisticamente nulo** (θ = 0,0031, IC cruza
zero; o placebo embaralhado dá quase o mesmo valor). **A história não está na média — está em
quando o efeito aparece:** ele é significativo **só nos anos de mercado aquecido (2022 e 2023, boom
pós-COVID)** e some no resto. A análise antiga (2023–2025) parecia significativa justamente porque
"pegou" o 2023 forte.

> Frase-síntese (atualizada): *"O prêmio do vendedor não é uma lei do mercado — ele emerge nas
> janelas de mercado aquecido. Na média de 8 temporadas, ele desaparece."*

## 7. Status atual

| Componente | Status |
|------------|--------|
| EDA e integração dos dados | ✅ Concluído |
| Feature engineering (clube + rede) | ✅ Concluído |
| Etapa 1 — modelo hedônico | ✅ Concluído |
| Etapa 2 — DML (ATE, placebos, CATE, IVB, sazonalidade) | ✅ Concluído (DML manual; econml indisponível) |
| **Expansão para 8 temporadas + reprocessamento** | ✅ Concluído ([changelog](reprocessamento_8temporadas.md)) |
| **Teste de robustez: tratamento defasado (C2)** | ✅ Concluído (agora significativo) |
| IVB comparável (CausalForestDML) | ⚠️ Pendente — feito via R-learner (econml não instala aqui) |
| Validação por PSM (prometida) | ❌ Não implementada |
| Enriquecimento com performance (FBref/StatsBomb) | ❌ Não feito |
| Tuning de hiperparâmetros (Optuna) | ❌ Não feito |
| Controle de C3 (causa comum) | ❌ Limitação reconhecida |

A lista completa de pendências, com impacto e esforço, está em [pendencias.md](pendencias.md).

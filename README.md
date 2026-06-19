# Ciência de Dados Aplicada ao Futebol — UFMG
## Efeito Dominó no Mercado de Transferências (Grupo 02)

Investigamos se clubes que acabaram de vender pagam um **"prêmio do vendedor"** nas compras
seguintes, separando o efeito causal da liquidez do mero porte financeiro do clube via um
pipeline causal de duas etapas (modelo hedônico → Double Machine Learning).

## Ambiente

- **Python 3.12.13** (ambiente em `.venv`, kernel Jupyter `tp-cdaf` / `python3`)
- Dependências fixadas em `requirements.txt`

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

> `econml` (CausalForestDML) **não** instala neste ambiente (Python 3.12 + NumPy 2.x → falha de
> build). O CATE/IVB usa um R-learner manual (Nie & Wager, 2021) como substituto ilustrativo.

## Ordem de execução dos notebooks

Os notebooks são **encadeados** — cada um consome o CSV gerado pelo anterior. Rodar nesta ordem:

| # | Notebook | Entrada | Saída |
|---|----------|---------|-------|
| 1 | `notebooks/feature_engineering.ipynb` | `output/transfers_enriched_consolidated.csv` | `output/transfers_modeling_ready.csv` |
| 2 | `notebooks/etapa1_hedonic_model.ipynb` | `transfers_modeling_ready.csv` | `output/transfers_etapa2_ready.csv` (Y = resíduo hedônico) |
| 3 | `notebooks/etapa2_double_ml.ipynb` | `transfers_etapa2_ready.csv` | `output/resultados_etapa2_ivb.csv` |
| 4 | `notebooks/etapa2_robustez.ipynb` | `transfers_etapa2_ready.csv` | testes de robustez |

`notebooks/exploratory_analysis.ipynb` é independente (EDA) e pode ser rodado a qualquer momento.

Execução não-interativa (reproduz todos os resultados):

```bash
for nb in feature_engineering etapa1_hedonic_model etapa2_double_ml etapa2_robustez; do
  .venv/bin/jupyter nbconvert --to notebook --execute --inplace \
    --ExecutePreprocessor.kernel_name=python3 "notebooks/$nb.ipynb"
done
```

## Principais achados

1. **A correlação mente:** o efeito bruto (OLS, θ≈0,012\*) encolhe para ~0,003 (n.s.) após o DML
   controlar o confundidor "clube rico". É o resultado mais robusto do trabalho.
2. **Efeito médio nulo, condicional ao regime:** o prêmio do vendedor só emerge em mercado
   aquecido (2022–2023, boom pós-COVID); na média de 8 temporadas é estatisticamente zero.
3. Análises exploratórias (mecanismo de sinalização, IVB) são geradoras de hipótese — ver as
   ressalvas nos respectivos notebooks.

Documentação completa em `docs/guia_completo_do_projeto.md`.

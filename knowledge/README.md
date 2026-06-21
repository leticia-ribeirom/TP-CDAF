# Base de Conhecimento — Efeito Dominó no Mercado de Transferências

Esta pasta documenta **o que o projeto é, o que já foi construído e o que ainda falta**.
Serve de fonte única para a construção do *storytelling*, da apresentação e do relatório final.

> Toda a documentação está em pt-br e baseada no que os **notebooks realmente produzem**
> (não nas previsões dos documentos de checkpoint, que descreviam a Etapa 2 ainda "em andamento").

## Índice

| Documento | Conteúdo |
|-----------|----------|
| [resumo.md](resumo.md) | Visão geral: proposta, pergunta de pesquisa, pipeline em duas etapas, números-chave e status atual. **Comece por aqui.** |
| [dados_e_features.md](dados_e_features.md) | Camada de dados: bases brutas, filtros, winsorização, features de clube e features de rede (grafo). |
| [hedonic_ml.md](hedonic_ml.md) | Etapa 1 — modelo hedônico (ML) que estima o "preço justo" e extrai o resíduo (prêmio). |
| [double_ml.md](double_ml.md) | Etapa 2 — Double Machine Learning, efeito causal (ATE), heterogeneidade (CATE) e o índice IVB. |
| [pendencias.md](pendencias.md) | Análise de lacunas: o que foi prometido na proposta vs. o que foi entregue, e o que falta. |
| [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md) | **Changelog** do re-run com 8 temporadas: o que mudou nos números e nos insights (o ATE deixou de ser significativo). |

## Mapa do repositório (referência rápida)

- `notebooks/exploratory_analysis.ipynb` — EDA e integração dos dados brutos
- `notebooks/feature_engineering.ipynb` — construção do dataset de modelagem
- `notebooks/etapa1_hedonic_model.ipynb` — modelo hedônico (Etapa 1)
- `notebooks/etapa2_double_ml.ipynb` — modelagem causal (Etapa 2; DML manual, sem econml)
- `notebooks/etapa2_robustez.ipynb` — robustez: teste de tratamento defasado (C2)
- `output/transfers_modeling_ready.csv` — saída da feature engineering
- `output/transfers_etapa2_ready.csv` — dataset com o resíduo hedônico (entrada da Etapa 2)
- `output/resultados_etapa2_ivb.csv` — ranking IVB por clube (190 clubes)
- `deliverables/` — proposta (TP1), checkpoint (TP2), roteiro e spec da Etapa 2

# Pendências — O que falta? Já é suficiente?

Este documento confronta **o que a proposta prometeu** com **o que foi entregue**, e prioriza o
que ainda falta. Serve para decidir se o trabalho está pronto para o relatório/apresentação ou se
algo precisa ser fechado antes.

> 🔄 **Atualizado após a expansão para 8 temporadas.** Algumas pendências foram fechadas (lag/C2),
> outras surgiram (econml/IVB, port da Etapa 2). Ver [reprocessamento_8temporadas.md](reprocessamento_8temporadas.md).
> **Mudança crítica:** com 8 temporadas o ATE deixou de ser significativo — a narrativa do
> relatório precisa migrar para a **heterogeneidade temporal** (efeito só em 2022–2023).

## TL;DR — É suficiente?

**Sim, o núcleo está completo e defensável.** O pipeline de duas etapas funciona, a estimativa
causal é válida (placebos limpos) e o IVB entrega um resultado original e tangível. **O que falta
é, em sua maioria, escopo prometido que a base de dados não permitiu** — e isso é legítimo desde
que assumido com honestidade no relatório.

**Há, porém, lacunas que merecem decisão consciente** (não apenas serem ignoradas):
1. **`econml` não instala** no ambiente (Python 3.14) → Etapa 2 roda via DML manual e o **IVB saiu
   por R-learner** (≠ CausalForestDML). Para um IVB comparável/definitivo, rodar em ambiente com
   econml (Colab/py3.12).
2. ~~`etapa2_double_ml.ipynb` não executa (importa econml)~~ ✅ **Portado para DML manual** —
   o notebook agora roda sem econml e reproduz os números (ATE, placebos, CATE/IVB via R-learner,
   sazonal). O CATE permanece como aproximação (R-learner) do CausalForestDML — ver item 1.
3. **PSM** foi prometido no checkpoint e **não existe** no código.
4. **Performance esportiva (FBref/StatsBomb)** era "essencial" na proposta e **não foi integrada**.
5. A linguagem de **"efeito dominó / janela de 30 dias"** não corresponde ao que foi modelado
   (liquidez **sazonal**), e precisa ser reescrita.

✅ **Fechado nesta rodada:** teste de robustez **C2 (tratamento defasado)** — implementado e agora
**significativo**; expansão para **8 temporadas**.

## 1. Proposta × Entrega

| Prometido na proposta/checkpoint | Entregue | Situação |
|----------------------------------|----------|----------|
| Janela de **30 dias** pós-venda ("efeito dominó na mesma janela") | Receita **por temporada** | 🔧 Reescopo — base não tem data diária |
| **Curva de decaimento temporal** θ(Δt) | θ por temporada (2017–2025) | 🔧 Substituído (honesto) |
| **PSM** ("clubes gêmeos") para validar C6 | — | ❌ **Não implementado** |
| Enriquecimento **FBref/StatsBomb** (gols, xG, assist., minutagem) | Só atributos básicos (idade, MV, posição, liga) | ❌ **Não feito** |
| **Tuning** de hiperparâmetros (Optuna) | Modelos com config padrão | ❌ Não feito (impacto baixo) |
| Controle de **C2** (causalidade reversa) | Testado via tratamento defasado | ✅ C2 enfraquecida (efeito persiste) |
| Controle de **C3** (causa comum) | — | ❌ Limitação assumida |
| **"Clubes predadores"** (lado vendedor) | IVB só do lado comprador | ⚠️ Parcial |
| Pergunta de **"probabilidade"** de sobrepreço | Intensidade (ATE) + heterogeneidade (IVB) | 🔧 Reformulada |
| Modelo hedônico + resíduo | RF, R² **0,762** (8 temporadas) | ✅ Concluído |
| **Double ML** (cross-fitting) | ATE, placebos, CATE/IVB (R-learner) | ✅ Concluído (DML manual) |
| **IVB** por clube | 190 clubes ranqueados (R-learner) | ✅ Concluído (ver ressalva econml) |

## 2. Status dos 6 confundidores

A proposta mapeou 6 confundidores. Onde cada um foi (ou não) tratado:

| ID | Confundidor | Estratégia prometida | Situação real |
|----|-------------|----------------------|---------------|
| C1 | Poder financeiro ("clube rico") | Efeitos fixos / controles | ⚠️ Parcial — W tem **proxies** (rede: pagerank/degrees; elenco; dummies de liga), mas **não** o volume financeiro direto (total_spend, n_buys) |
| C2 | Causalidade reversa | Ordenar eventos por data | ⚠️ **Testado via lag** (tratamento defasado): efeito persiste e significativo → C2 enfraquecida |
| C3 | Causa comum / planejamento | Contexto esportivo (UCL, técnico) | ❌ Não tratado (sem esses dados) |
| C4 | Sazonalidade / deadline | Efeitos fixos de janela | ✅ Dummies de temporada em W |
| C5 | Inflação de mercado | Ajuste pela inflação da janela | ✅ Via FE de temporada (7 dummies) + dummies de liga |
| C6 | Viés de seleção | **PSM** ("clube gêmeo") | ⚠️ Proxies em W, mas **PSM não feito** |

## 3. Pendências priorizadas (impacto × esforço)

### Alta prioridade (decidir antes do relatório)
- **[Narrativa] Reescrever o enquadramento "efeito dominó / 30 dias".** O tratamento é liquidez
  **sazonal**. Sem isso, o relatório promete algo que o modelo não faz. *Esforço: baixo.*
- **[Decisão] PSM: implementar OU declarar explicitamente como não realizado.** Hoje está em um
  limbo (prometido no checkpoint, ausente no código e não citado nas limitações). *Esforço:
  médio se implementar; baixo se assumir como limitação.*
- **[Narrativa] Liderar pela heterogeneidade temporal, não pelo ATE.** Com 8 temporadas o ATE é
  **não significativo**; a manchete honesta é "o prêmio emerge só em mercado aquecido (2022–2023)".
  *Esforço: baixo (é decisão de storytelling).*

### Média prioridade (fortaleceriam o trabalho)
- **[Modelo] Integrar performance (FBref/StatsBomb)** no hedônico → eleva o R² e "limpa" o
  resíduo. Era "essencial" na proposta. *Esforço: alto (entity matching entre bases).*
- **[Robustez] Testes ainda não feitos:** especificações alternativas de D (`n_sales`, flag
  `big_sale`), subgrupos por tier de liga/posição. *(O lag/C2 já foi feito.)* *Esforço: médio.*
- **[Análise] "Clubes predadores" (lado vendedor)** — simétrico ao IVB; fecharia uma promessa da
  proposta. *Esforço: médio.*

### Baixa prioridade (nice to have)
- **[Modelo] Tuning com Optuna** — ganho marginal (RF já generaliza bem). *Esforço: médio.*
- **[IVB] Reprocessar com `CausalForestDML`** (ambiente com econml) para um ranking comparável ao
  desenho original. *Esforço: baixo se houver econml.*

## 4. Riscos a vigiar

- **ATE não significativo:** com 8 temporadas o efeito médio cruza zero e o placebo embaralhado
  fica quase do tamanho do ATE. A narrativa **não pode** sustentar "existe um prêmio causal médio";
  o achado é a **heterogeneidade temporal** (efeito só em 2022–2023).
- **C1 controlado por proxies:** o W não tem o volume financeiro direto (total_spend/n_buys) — o
  controle de "clube rico" depende de centralidade na rede + elenco + dummies de liga.
- **IVB instável:** ranking via R-learner é dominado por outliers de poucas observações (Nîmes) e
  mudou totalmente vs. a versão anterior — tratar como ilustrativo.
- **Resíduo possivelmente contaminado:** como o hedônico não usa performance, parte do "preço
  justo" fica por explicar e pode vazar para o resíduo (Y).

## 5. Recomendação

Para a **entrega final**, o caminho de menor risco e maior retorno é:
1. **Não abrir novas frentes de modelagem** (PSM/FBref são caros e o núcleo já responde à
   pergunta). Tratá-los como **trabalho futuro** declarado.
2. **Fechar as lacunas narrativas** (reescopo do "dominó", liderar pela heterogeneidade) — baixo
   esforço, alto impacto na credibilidade.
3. **Rodar os testes de robustez baratos** que já estão especificados no docx (D alternativo, lag,
   subgrupos) — fortalecem muito a seção causal com esforço médio.
4. **Decidir conscientemente sobre o PSM** — implementar a versão simples (sanity check) ou
   assumi-lo como limitação. O pior cenário é o silêncio atual.

> Próximo passo natural: a partir desta base, construir o **storytelling** (espinha narrativa) que
> alimenta apresentação e relatório.

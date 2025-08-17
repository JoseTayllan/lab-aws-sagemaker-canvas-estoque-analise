# 📊 Previsão de Estoque Inteligente na AWS com SageMaker Canvas

> **Projeto do Lab DIO** – Documentação completa com passos, decisões, resultados, estudo pessoal e próximos passos. 

---

## Sumário

- [Visão Geral](#visão-geral)
- [Arquitetura e Fluxo](#arquitetura-e-fluxo)
- [Dados](#dados)
  - [Origem](#origem)
  - [Dicionário de Dados](#dicionário-de-dados)
  - [Preparação & Qualidade](#preparação--qualidade)
- [Modelagem no SageMaker Canvas](#modelagem-no-sagemaker-canvas)
  - [Configurações do Modelo](#configurações-do-modelo)
  - [Treinamento](#treinamento)
  - [Avaliação](#avaliação)
  - [Interpretação](#interpretação)
- [Previsões e Uso no Negócio](#previsões-e-uso-no-negócio)
- [Estudo & Compreensão (reflexões pessoais)](#estudo--compreensão-reflexões-pessoais)
- [Experimentos (log)](#experimentos-log)
- [Custos, Segurança e Governança](#custos-segurança-e-governança)
- [Limitações e Próximos Passos](#limitações-e-próximos-passos)
- [Reprodutibilidade (passo a passo)](#reprodutibilidade-passo-a-passo)
- [Checklist para Submissão na DIO](#checklist-para-submissão-na-dio)
- [Licença](#licença)

---

## Visão Geral

**Objetivo**: prever demanda/saída de estoque por produto e período para apoiar decisões de reposição, reduzir ruptura e otimizar capital de giro.

- **Problema**: Previsão de séries temporais (regressão/forecasting) por SKU (e opcionalmente por loja).
- **Ferramenta**: **Amazon SageMaker Canvas** (ML no-code).
- **Resultado-chave**: previsões por horizonte **H = ** (ex.: 4 semanas) com erro **MAPE = ****%** e **RMSE = **.

## Arquitetura e Fluxo

```
[Dataset CSV/Parquet] → [S3] → [SageMaker Canvas]
                                     │
                        [Treinamento + Avaliação]
                                     │
                           [Batch Predictions]
                                     │
                              [S3 / CSV export]
                                     │
                 [BI (QuickSight) / Planilhas / ERP/OMS]
```

- **Entrada**: arquivo(s) em CSV/Parquet no S3 ou upload direto no Canvas.
- **Saída**: arquivo de previsões (CSV) por SKU/loja/data, com intervalos de confiança (quando disponíveis).

## Dados

### Origem

- **Dataset escolhido**: `<dataset-500-curso-sagemaker-canvas-dio.csv>`
- **Granularidade temporal**: `<diária | semanal | mensal>`
- **Chaves de série**: `<ex.: sku_id, loja_id>`
- **Janela de histórico**: `<ex.: 24 meses (2023-01 a 2024-12)>`
- **Horizonte de previsão (H)**: `<ex.: 8 semanas>`

### Dicionário de Dados

| Coluna        | Tipo     | Exemplo    | Descrição                        |
| ------------- | -------- | ---------- | -------------------------------- |
| `date`        | data     | 2024-07-01 | Período de referência            |
| `sku_id`      | string   | 123-ABC    | Identificador do produto         |
| `store_id`    | string   | SP-01      | (Opcional) Loja ou canal         |
| `qty_sold`    | numérico | 87         | Quantidade vendida/saída (alvo)  |
| `<feature_1>` | `<tipo>` | `<ex>`     | Promoções, feriados, preço, etc. |

> **Alvo (y)**: `qty_sold`

### Preparação & Qualidade

- **Tratamento**: valores ausentes, outliers, deduplicação, datas inválidas.
- **Sazonalidade/feriados**: `<incluiu/ não incluiu>` (ex.: calendário de feriados nacionais).
- **Features externas**: `<preço, promo, lead_time, estoque anterior, etc.>`
- **Split**: treino vs. validação ``.

## Modelagem no SageMaker Canvas

### Configurações do Modelo

- **Tipo**: Previsão de série temporal / Regressão.
- **Coluna alvo**: `qty_sold`
- **Coluna de data**: `date` | **Frequência**: `<diária/semanal>`
- **Chaves de agrupamento**: `sku_id` (e `store_id`, se houver)
- **Horizonte (H)**: `<preencha>`
- **Build**: `<Quick/Standard>` (Canvas oferece níveis de build; escolha conforme tempo/qualidade desejada)

### Treinamento

- Importar dataset no Canvas → Selecionar problema → Definir alvo, data, chaves e horizonte → **Build**.
- Aguardar conclusão e abrir a aba de **Avaliação**.

### Avaliação

**Rodada 001 – Classificação (alvo = **``**)**

> O Canvas configurou o problema como **classificação multiclasse**, tentando prever o valor da coluna `DIA` (datas) em vez de prever quantidade. Como consequência, as métricas abaixo são de **classificação** e não refletem a qualidade de uma previsão de estoque.

| Métrica                            | Valor      |
| ---------------------------------- | ---------- |
| **Accuracy (Optimization metric)** | **8%**     |
| **Average F1**                     | **7.001%** |
| **Average Precision**              | **7.181%** |
| **Average Recall**                 | **8%**     |
| **Average AUC-ROC**                | **N/A**    |

**Como interpretar:**

- **Accuracy 8%**: em média, o modelo acerta a **classe** (um dia específico) em 8% dos casos. O Canvas mostra que sempre prever o rótulo mais frequente (ex.: *2024-01-10*) acerta **≈9,091%** dos exemplos — ou seja, o modelo está **no nível do baseline**.
- **Precision/Recall/F1 \~7–8%**: como há **muitas classes** (vários valores de `DIA`) e forte **diluição** das frequências, a média por classe fica muito baixa.
- **AUC-ROC N/A**: o Canvas não reporta AUC-ROC para certos cenários de **multiclasse**.

> **Conclusão:** essas métricas sugerem **configuração incorreta do problema** para o objetivo de **prever estoque/demanda**. Precisamos trocar para **Séries Temporais/Regressão** com alvo numérico.

**O que esperamos após a correção (forecast):**

- Métricas de erro **regressão/forecast** (ex.: **MAPE, RMSE, MAE, R²**).
- Série temporal com ``**/**`` e alvo **quantitativo** (ex.: `QUANTIDADE_ESTOQUE` ou `QTD_SAIDA`).

### Interpretação

**Impacto de colunas (Canvas) – execução atual**

> *Atenção: como o alvo estava incorreto (**`DIA`**), a importância abaixo reflete o que influencia o ****dia**** previsto, e ****não**** o que explica a ****quantidade****. Use apenas como diagnóstico.*

| Feature              | Importância |
| -------------------- | ----------- |
| `QUANTIDADE_ESTOQUE` | **79.89%**  |
| `ID_PRODUTO`         | **16.072%** |
| `FLAG_PROMOCAO`      | **4.038%**  |

**Leituras rápidas das dispersões (prints do Canvas):**

- `QUANTIDADE_ESTOQUE` mostra forte variação de impacto ao longo do eixo (pontos concentrados e saltos em valores altos), o que indica **não linearidades**.
- `ID_PRODUTO` captura diferenças por produto (efeitos fixos de SKU).
- `FLAG_PROMOCAO` tem impacto pequeno nesta configuração.

**Correção recomendada para o objetivo de *****prever estoque***

1. **Tipo de problema**: selecionar **Time series / Regression** no Canvas.
2. **Alvo (y)**: `QUANTIDADE_ESTOQUE` (ou a coluna de **saída/venda/demanda**).
3. **Coluna de data**: usar `DIA` (ou equivalente) configurada como **timestamp**; defina a **frequência** (diária/semanal).
4. **Chaves de série**: `ID_PRODUTO` (e `LOJA` se houver) para criar **múltiplas séries por SKU/loja**.
5. **Atributos explicativos**: manter `FLAG_PROMOCAO` e outros (preço, feriados, etc.).
6. **Horizonte (H)**: definir (ex.: 4–8 períodos).
7. **Build**: começar com *Quick*, depois *Standard* para qualidade.

> Após essa mudança, preencha a seção de **Avaliação (forecast)** com **MAPE/RMSE/MAE/R²** e atualize os gráficos de importância.

## Previsões e Uso no Negócio

- **Export**: Gere *Batch predictions* no Canvas e **exporte CSV** para S3/Download.
- **Indicadores operacionais**:
  - **Ponto de Pedido (ROP)**: \(ROP = d_L + SS\)
  - **Estoque de Segurança (SS)**: \(SS = z \cdot \sigma_L \cdot \sqrt{L}\)
  - **Parâmetros**:
    - \(d_L\): demanda média no lead time **(use a previsão)**
    - \(\sigma_L\): desvio-padrão da demanda no lead time **(use erros históricos)**
    - \(z\): fator do nível de serviço (ex.: 1,64 ≈ 95%)
- **Como aplicar**: para cada SKU/loja, calcule **ROP** e **SS**; quando **estoque atual ≤ ROP**, gere ordem de compra **Q** (ex.: política \((s, S)\) ou EOQ).

## Estudo & Compreensão (reflexões pessoais)

> Escreva com suas palavras (aprendizado é parte da avaliação!). Exemplos de tópicos:

- **O que aprendi sobre séries temporais**: `<ex.: tendência, sazonalidade, efeito calendário>`
- **Qualidade de dados importa**: `<ex.: outliers de rupturas e lançamentos de novos SKUs distorcem>`
- **Trade-offs**: `<ex.: modelo rápido vs. modelo mais preciso; horizonte longo aumenta erro>`
- **Impacto no negócio**: `<ex.: redução de ruptura e melhor giro de estoque>`

## Experimentos (log)

| ID  | Data | Mudança       | Objetivo              | Resultado |
| --- | ---- | ------------- | --------------------- | --------- |
| 001 |      | Base          | Baseline              | MAPE %    |
| 002 |      | + Feriados    | Capturar sazonalidade | MAPE %    |
| 003 |      | + Promo/Preço | Sensibilidade a preço | MAPE %    |

> Dica: mantenha este quadro ao evoluir o modelo (ajuda a justificar decisões).

## Custos, Segurança e Governança

- **Custos**: o Canvas utiliza recursos gerenciados; monitore consumo e **pare ambientes ociosos**.
- **Segurança/IAM**: princípio de **menor privilégio** (leitura no bucket de dados, gravação apenas na pasta de saída).
- **Governança**: versionar datasets, registrar métricas, e armazenar *artefatos* (CSV de previsão) com controle de acesso.

## Limitações e Próximos Passos

- **Limitações**:
  - Dados com pouca história por SKU dificultam previsões.
  - Quebras de série (lançamentos, descontinuações) exigem tratamento.
- **Próximos passos**:
  - Enriquecer features (clima, calendário promocional, preço concorrente).
  - Separar clusters de SKUs (ABC/XYZ) e modelos por grupo.
  - Publicar previsões em **QuickSight** e automatizar via pipeline.

## Reprodutibilidade (passo a passo)

1. **Preparar dados**: CSV com colunas `date`, `sku_id`, `qty_sold`, etc.
2. **Armazenar** no **S3** ou **upload** direto no Canvas.
3. **Abrir o SageMaker Canvas** → **Create model** → **Time series/Regression**.
4. **Configurar** alvo, data, chaves e horizonte.
5. **Build** (Quick/Standard) e aguardar.
6. **Avaliar** métricas (MAPE, RMSE, MAE, R²) e importância de features.
7. **Gerar Previsões** (Batch) e exportar CSV.
8. **Aplicar** fórmulas de ROP/SS e integrar ao processo (planilha/BI/ERP).

## Checklist para Submissão na DIO

-

## Licença

Este projeto está sob a licença **MIT**. Sinta-se à vontade para usar e adaptar, citando a fonte.

---

### Anexos (opcional)

- Prints da tela do Canvas (Avaliação, Importância, Previsões).
- Amostra do CSV de previsão (`outputs/forecast_sample.csv`).


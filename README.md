# Previsão de recompra de clientes em 60 dias (Online Retail Dataset)

Análise de dados e modelagem preditiva sobre as vendas de um varejo online do Reino Unido para responder:

> **Quais clientes têm maior probabilidade de comprar novamente nos próximos 60 dias e como usar essa informação para priorizar ações comerciais?**

O trabalho tem duas partes: uma **análise exploratória do negócio** (receita, cancelamentos, produtos, países e segmentação RFM) e uma **modelagem preditiva** de recompra, com comparação de modelos.

## Dados

- **Fonte:** [Online Retail Dataset, UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/352/online+retail)
- **Período:** 01/12/2010 a 09/12/2011
- **Tamanho original:** 541.909 registros de transações de uma empresa de varejo do Reino Unido
- **Campos principais:** `InvoiceNo`, `StockCode`, `Description`, `Quantity`, `InvoiceDate`, `UnitPrice`, `CustomerID`, `Country`

### Limpeza

- Remoção de linhas sem `CustomerID` (não podem ser associadas a um cliente)
- Separação dos pedidos cancelados (códigos iniciados em `C`) para análise própria e remoção deles da base principal
- Remoção de quantidades e preços não positivos
- Remoção de outliers de quantidade e preço unitário (método IQR)
- **Base final:** 382.861 linhas, 17.626 pedidos e 4.257 clientes

## Principais resultados da análise exploratória

| Indicador | Valor |
| --- | --- |
| Ticket médio | £ 338,77 |
| Itens por pedido | 181,39 |
| Frequência de compra | 4,14 pedidos por cliente |
| Taxa de recorrência (clientes com mais de um pedido) | 65,09% |
| Taxa de cancelamento geral | 17,17% (3.654 pedidos cancelados em 21.280 pedidos) |
| Preço médio dos itens vendidos | £ 2,91 |
| Itens únicos | 3.824 |

Outros achados:

- Os meses com menos pedidos vão de dezembro a abril; novembro de 2011 teve o maior volume (2.547 pedidos).
- Fora do Reino Unido, os países com maior receita são Alemanha (£ 184.288,57), Irlanda/EIRE (£ 165.157,10) e França (£ 157.013,74).
- 1.013 itens foram vendidos em apenas um país, e 23 itens custam menos de 1 € e vendem 5 unidades ou menos, com risco de prejuízo em estoque.
- Os itens mais comprados incluem "ASSORTED COLOUR BIRD ORNAMENT" (20.002 unidades) e "WHITE HANGING HEART T-LIGHT HOLDER" (18.618 unidades).

### Segmentação RFM (Recência, Frequência, Valor monetário)

| Segmento | Clientes |
| --- | --- |
| Melhores clientes | 2.225 |
| Clientes potenciais | 753 |
| Clientes leais | 741 |
| Clientes em risco | 538 |

## Modelagem preditiva

### Definição do problema

- **Janela de observação:** transações até 10/10/2011 (269.334 transações), usadas para calcular as variáveis dos clientes
- **Janela de previsão:** os 60 dias seguintes, de 10/10/2011 a 09/12/2011 (113.527 transações)
- **Alvo:** o cliente voltou a comprar na janela de previsão (1) ou não (0)
- **Base de modelagem:** 3.654 clientes, dos quais 1.756 recompraram (48,1%) e 1.898 não
- **Separação:** 80% treino (2.923) e 20% teste (731), estratificada, com `random_state=42`
- As variáveis são calculadas **somente com dados da janela de observação**, para evitar vazamento de informação do futuro.

### Três etapas de teste

1. **RFM:** Recência, Frequência e Valor monetário.
2. **Variáveis expandidas (43 colunas):** RFM + quantidade total comprada, diversidade de produtos, taxa de cancelamento do cliente, média de dias entre pedidos e país principal (one-hot), com uma seleção de variáveis a partir dos coeficientes da regressão logística.
3. **Variáveis agregadas (5 colunas):** recência, frequência, valor, quantidade total e itens únicos.

### Comparação dos modelos (conjunto de teste, classe "recompra")

| Etapa | Modelo | Acurácia | Precisão | Recall | F1 | ROC AUC |
| --- | --- | --- | --- | --- | --- | --- |
| RFM | Regressão logística | 0,6676 | 0,6929 | 0,5527 | 0,6149 | 0,7267 |
| RFM | Random Forest | 0,6197 | 0,6159 | 0,5527 | 0,5826 | 0,6782 |
| Expandida | Regressão logística | 0,6703 | 0,6978 | 0,5527 | 0,6169 | 0,7308 |
| Expandida | Random Forest | 0,6539 | 0,6655 | 0,5613 | 0,6090 | 0,7041 |
| Expandida | LightGBM | 0,6361 | 0,6412 | 0,5499 | 0,5920 | 0,6935 |
| Expandida | Árvore de decisão | 0,5677 | 0,5519 | 0,5299 | 0,5407 | 0,5663 |
| Expandida, seleção de variáveis (14) | Regressão logística | 0,6580 | 0,7149 | 0,4786 | 0,5734 | 0,7173 |
| Agregada (5) | **Regressão logística** | **0,6689** | 0,6912 | 0,5613 | **0,6195** | **0,7322** |
| Agregada (5) | LightGBM | 0,6416 | 0,6459 | 0,5613 | 0,6006 | 0,6874 |
| Agregada (5) | Random Forest | 0,6265 | 0,6318 | 0,5328 | 0,5781 | 0,6806 |
| Agregada (5) | XGBoost | 0,6183 | 0,6161 | 0,5442 | 0,5779 | 0,6631 |
| Agregada (5) | Árvore de decisão | 0,5431 | 0,5246 | 0,5157 | 0,5201 | 0,5420 |

Todas as etapas usam a mesma divisão de clientes entre treino e teste, então os números são comparáveis entre si.

### Conclusão da modelagem

- O **melhor modelo** foi a **regressão logística com as 5 variáveis agregadas**: ROC AUC de 0,7322, F1 de 0,6195 e acurácia de 0,6689. Ela empata, na prática, com a regressão logística das 43 variáveis expandidas (ROC AUC de 0,7308), usando muito menos informação.
- Entre as variáveis de comportamento, os maiores coeficientes da regressão expandida foram **taxa de cancelamento** (0,2544) e **frequência de compras** (0,2014). Os coeficientes de países pequenos (como Suíça e Austrália) são altos, mas vêm de poucos clientes.
- Com as 14 variáveis selecionadas pelos coeficientes, a precisão sobe para 0,7149, ao custo de um recall menor (0,4786).
- Modelos mais complexos não superaram a regressão logística neste volume de dados.

## Como usar o resultado no negócio

- **Priorizar clientes** em listas de e-mail e mensagens, com base na probabilidade de recompra.
- **Personalizar ofertas** com os itens mais comprados pelos clientes de maior valor.
- **Controlar compras de estoque**, já que há itens que praticamente não vendem.
- **Priorizar produtos por país**, a partir dos itens mais vendidos em cada mercado.
- **Investigar cancelamentos e a sazonalidade** entre dezembro e abril.

## Decisões metodológicas

- Exclusão dos pedidos cancelados da base de modelagem (analisados à parte).
- Exclusão dos clientes sem identificador.
- Teste de vários modelos e de diferentes conjuntos de variáveis.
- Exclusão da geolocalização (país) do conjunto de variáveis selecionadas, devido ao desbalanceamento dos dados.

## Limitações e próximos passos

- O desempenho é **moderado** (ROC AUC de cerca de 0,73) e foi medido em **uma única divisão treino/teste** de 731 clientes. Validação cruzada e ajuste de hiperparâmetros são o passo natural.
- A base cobre um único ano, e a janela de previsão inclui o período de fim de ano, o que pode influenciar a recompra.
- As variáveis não foram padronizadas na regressão logística, então os coeficientes não são diretamente comparáveis entre si.
- Próximos passos: validação cruzada, calibração das probabilidades, novas variáveis de comportamento e teste em outro período.

## Tecnologias

Python, pandas, NumPy, matplotlib, seaborn, scikit-learn (Regressão Logística, Random Forest, Árvore de Decisão), LightGBM e XGBoost.

## Como executar

1. Baixe o arquivo `Online Retail.xlsx` no [repositório da UCI](https://archive.ics.uci.edu/dataset/352/online+retail) e coloque-o na mesma pasta do notebook.
2. Instale as dependências (versões testadas em `requirements.txt`):
   ```bash
   pip install -r requirements.txt
   ```
3. Abra o `Online_Retail_Dataset.ipynb` e execute todas as células, em ordem (*Run All*).

## Organização do notebook

- **Parâmetros** reunidos numa célula no início (janela de previsão, semente, proporção de teste, limiar de seleção de variáveis).
- **Funções auxiliares** para os passos que se repetem: contagem de pedidos por período, gráficos, cálculo de RFM, treino e avaliação de modelos (relatório, métricas, curva ROC e matriz de confusão) e tabela comparativa.
- O notebook roda **do início ao fim, na ordem**, sem depender de células executadas fora de sequência.

## Autora

**Júlia Roberta Quoos Alves**: engenheira de dados e mestranda em Ciência da Computação (UFSM).
[LinkedIn](https://www.linkedin.com/in/ohtopico) · [Site](http://ohtopico.com.br)

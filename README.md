# Modelo de Detecção de Fraude em Transações de Cartão
![label](https://img.shields.io/badge/Python-3.10+-blue?logo=python&logoColor=white)

# Índice 
* [Contexto do Problema](#contexto-do-problema)
* [Objetivos](#objetivos)
* [Como Executar](#como-executar)
* [Decisões Técnica](#decisões-técnicas)
* [Resultados](#resultados)
* [Plano de Deply e Monitoramento](#plano-de-deploy-e-monitoramento)


## Contexto do Problema
Neste projeto foi desenvolvido um modelo de detecção de fraude em transações de cartão.

## Objetivos
* Realizar uma Análise Exploratória de Dados 
* Criar features que traduzam regras de negócio e anomalias de segurança
* Treinar e comparar modelos de Machine Learning 
* Propor uma estratégia de produtização do modelo.

## Como Executar 
Para executar o projeto, 

Você pode baixar o dataset [aqui](https://drive.google.com/file/d/1Ms99L-UZMABlCBKkE6HfhF9uEHc0wkAT/view)

Após baixar o projeto, você pode abrir com Jupyter Notebook ou com Google Colab.

Caso opte por Jupyter Notebook, instale as bibliotecas necessárias usando o seguinte código:

~~~bash
pip install -r requirements.txt
jupyter notebook Teste-VOM.ipynb
~~~

Caso opte pelo Google Colab, as bibliotecas já vem instaladas por padrão. 

## Decisões Técnicas 
### Média e Mediana
Para comparar as médias e medianas de casos de fraude e não fraude, optei por dividir o dataset pela classe ``fraud``. O desbalanceamento do dataset faria com que a média e mediana de casos de fraudes, fosse "engolido" pelo pelos casos de não fraude. Então optei por essa divisão para termos uma medida mais precisa. 

### Variável ``risk_score``
Na parte de hipoteses, mostrei que encontrei um padrão estranho quanto a combinação da variávies categóricas. Foi tabelado todas as combinações e dado um nome para elas, junto com suas taxas de fraude:


|``repeat_retailer``|``used_chip``|``used_pin_number``|``online_order``| tipo de compra| Taxa de Fraude|
|:-:|:-:|:-:|:-:|:--:|:--:|
|0|0|0|0|Contacless|10.22%|
|0|0|0|1|Online|11.49%|
|0|0|1|0|Contactless + senha|0%|
|0|0|1|1|Online + autenticação|0.16%|
|0|1|0|0|Chip + assinatura|0.12%|
|0|1|0|1|Token + online|11.62%|
|0|1|1|0|Inserção + senha|0%|
|0|1|1|1|Anomalia|0.21%|
|1|0|0|0|Contactless|0.79%|
|1|0|0|1|Online|16.62%|
|1|0|1|0|Contactless + senha|0%|
|1|0|1|1|Online + autenticação|0.68%|
|1|1|0|0|Chip + assinatura|0.82%|
|1|1|0|1|Token + online|10.37%|
|1|1|1|0|Inserção + senha|0.01%|
|1|1|1|1|Anomalia|0.01%|

Transações online de clientes novos sem autenticação tem 2x mais risco que clientes recorrentes. Dado essas informações, optei pela criação de uma variável categórica para classificar o grau de risco de cada combinação.

* 4 (Crítico) 
* 3 (Alto)  
* 2 (Médio)
* 1 (Baixo) 



Essa nova variável vai ajudar nosso primeira escolha de modelo, o ``Logistic Regression
``
### Interpretação de Balanceamento 
Verificando a proporção de fraude e não fraude do dataset, tivemos os resultados:

* não frude = 91,26%
* fraude = 8,74%

Temos um desbalanceamento considerado moderado no dataset. Essa informação influência diretamente na escolha entre ``class_weight`` ou ``SMOTE``.

Como nosso dataset não tem um desbalanceamento alto, optei por não usar SMOTE, já que ele pode atrapalhar. Nosso datset possui 1 milhão de linhas, o uso de SMOTE aumentaria demais o tamanho do conjunto de treino sem uma necessidade real, atapalhando o custo operacional do nosso modelo. Usar ``class_weight='balanced'`` ajusta os pesos das classes proporcionamente à sua frequência, sem necessidade de criar dados sintéticos via SMOTE, preservando a distribuição real. 

## Resultados 

Como esperado, **Random Forest** teve desemepenho superior ao **Logistic Regreesion**

Podemos comparar os resultados através das métricas atingidas pelos modelos:

 Modelo | AUC | Precision | Recall | F1-Score | Accuracy
:---: | :---: | :---: | :---: | :---: | :---:  
Logistic Regresion |0.9439|0.5816|0.9534|0.7225|0.9360 
Random Forest |0.9999|1.0|0.9999|0.9999|0.9999 

Com isso, podemos concluir que a escolha ideal de modelo, seria a de **Random Forest**

## Plano de deploy e monitoramento

Temos um modelo que prever fraudes em transações com cartão, então temos um modelo que precisa está no ar o tempo todo. Por ter essas características de precisar está no ar o tempo todo, podemos subir esse modelo em Docker. Para fazer isso, podemos usar uma API que recebe os dados em formato JSON e retorna a saída do modelo. Podemos fazer várias instâncias da API com vários modelos no ar para aguentar o tranco de muitas requisições ao mesmo tempo. 

O modelo precisa funcionar em tempo real, cada transação dever ser avaliada imediatamente antes de ser aprovada ou negada. Então o plano precisa garantir alguns pontos: ter disponibilidade, baixa latência e coerência entre o ambiente de treino e produção.

1. Disponibilização do Modelo como Serviço:

     O modelo deve ser um serviço que recebe informações de transação, aplica o mesmo pré processamento usado no treino, calcular a probabilidade de fraude e retornar a decisão. Isso permite que qualquer sistema da empresa envie transações para serem avaliadas em tempo real. Podemos fazer isso criando uma API  que recebe os dados da transação em formato JSON e retorna a saída do modelo. E para o modelo rodando em tempo real, é interessante dockerizar.

2. Escalabilidade e Alta Disponibilidade:

    Como a quantidade de transações pode variar muito, o ambiente deve ter várias instâncias do modelo rodando ao mesmo tempo, para suportar um grande número de requisições.

3. Reprodutibilidade:

    O ambiente em produção precisa ter o mesmo pré processamento, o mesmo conjunto de features e a mesma versão do modelo usada nos testes.

4. Monitoramento:

    O modelo não deve ser dado como finalizado, mesmo após o deploy. É preciso garantir que o modelo mantenha o desempenho (acuracia, recall de classe fraude, etc). Outro ponto importante, é a mudança de estratégia de fraudadores, que pode causar uma mudança de comportamento dos dados. Se for detectado qualquer quede significativa de performace, o modelo precisa ser reavaliado.

# MVP SPRINT: ENGENHARIA DE DADOS
**Aluna:** Leticia Quintanilha

**Plataforma de nuvem adotada:** Google Cloud (Big Query)

**Tema:** Mobilidade Urbana: Sistema de ônibus do RJ

O sistema de ônibus na cidade do Rio de Janeiro é operado por diferentes empresas, atuando por meio de 5 consórcios. Após a quebra financeira de diversos operadores e o quase colapso do sistema durante a pandemia, a Prefeitura passou a subisidiar parte do serviço - mediante regras definidas em um acordo judicial firmado em junho de 2022.
Desde então, muitas têm sido as discussões em torno do cumprimento das regras por parte dos operadores assim como do gerencimento dos dados feito pela Prefeitura, estabelecendo uma verdadeira queda de braços para o pagamento do subsídio. 

**Objetivo:**  
Diante desse contexto, esse trabalho tem como objetivo criar um pipeline para a formação de um banco de dados sobre o sistema de ônibus do Rio de Janeiro, que facilite a realização de consultas para melhor entendê-lo. Assim, a ideia é tornar os alguns dados sobre os ônibus e o subsídio mais claros e acessíveis, especialmente em relação à perspectiva das empresas - o que não é facilitado na organização atual da base de dados pública da Prefeitura.  

**Questões:**  
No acordo judicial estabelecido, o pagamento do subsídio ocorre entre Prefeitura e consórcios, cada um deles composto por diversas empresas operando as diferentes linhas. As regras de conformidade para o pagamento ficam, por sua vez, relacionadas ao cumprimento de parâmetros mínimos determinados para cada linha (como cumprir o trajeto planejado, começar e terminar as viagens na origem e destino estabelecidos e manter a transmissão por gps durante as viagens, etc.).  
As questões então começam a surgir com o fato de que uma linha pode ser compartilhada por diferentes empresas, criando muitas vezes um cenário de disputas dentro dos consórcios para a distribuição interna do subsídio e das respectivas penalidades. Uma base de dados que organize de forma clara as responsabilidades e operação das empresas permitiria então mapear de forma mais clara **quais empresas estão cumprindo melhor ou pior sua parte no contrato, quais as principais dificuldades de cada uma e quais as dificuldades em comum,** permitindo também endereçar os problemas de forma mais eficiente.  
Assim, também como pesquisadora da área de mobilidade, gostaria de poder, ao final do trabalho, responder através dos dados no banco criado na nuvem as seguintes perguntas: 

- Quantas viagens são feitas em média,  em dias úteis, por cada empresa?
- Qual a relação média de veículos por linha operada em cada empresa?
- Qual o percentual de quilômetros rodados por cada empresa em cada consórcio nos últimos 6 meses de dados disponíveis?
- Qual empresa apresentou menor percentual de viagens não subsidiadas (com informidades) no último mês de dados disponíveis?
- Qual a regra para o subsídio tem maior percentual de violação em cada empresa?

## OS DADOS
**Fontes dos dados:** Os dados que irão compor o banco proposto estão disponíveis de forma pública na base de dados aberta da Prefeitura do Rio de Janeiro - o data.Rio. A maior parte deles se encontra em um Datalake já no Big Query da Google Cloud (pode ser acessado [neste link](https://www.data.rio/documents/PCRJ::transporte-rodovi%C3%A1rio-subs%C3%ADdio-dos-%C3%B4nibus-sppo/about)), sendo complementados por outros dados obtidos no formato .csv também disponíveis no [data.rio](https://www.data.rio/).

### COLETA DOS DADOS
Uma vez que a maior parte dos dados já se encontram no BigQuery, o caminho mais simples e imediato para compor o Data Warehouse desejado seria a coletar as informações de interesse por meio de queries SQL no espaço de consultas disponibilizado pela própria ferramenta, populando assim as tabelas para posterior consulta. No entanto, para fins didáticos deste estudo, trabalharemos com duas outras possibilidades de coleta, para que possa ser também utilizada a ferramenta Google Data Fusion nos processos de ETL. Além disso, para a utilização de outros arquivos disponíveis no data.rio em formato .csv ou compostos a partir de elaboração própria, faz-se necessário outro procedimento além das consultas no Big Query.

#### 1) Coleta por download de arquivos .csv
Nesse caso trabalharemos com alguns arquivos cuja fonte disponibiliza os dados em formato .csv e também tabelas de elaboração própria, resultantes de outros estudos previamente realizados.

Para a coleta dos dados, além do download realizado diretamente no site e separação dos arquivos próprios, os dados "brutos" em formato .csv foram armazenados em um *bucket* criado no Google Cloud Storage (GCS) para posterior trabalho de ETL.
![arquivos no bucket](prints\load_arquivos_bucket.png)

#### 2) Coleta a partir do Big Query
Para os dados disponíveis no Big Query o procedimento de coleta foi feito utilizando a ferramenta *Big Query Execute*, dentro do Google Data Fusion. Assim, através de queries SQL, foram criadas tabelas com os dados "raw" em um dataset dentro do Big Query próprio. 
Foram então empregadas as seguintes queries para a carga dos dados:

``` sql
-- coleta dos dados das linhas planejadas pela Prefeitura

CREATE TABLE data-science-puc.onibus_raw.viagem_planejada_onibus AS (
SELECT
  tipo_dia,
  servico,
  vista,
  consorcio,
  sentido,
  distancia_planejada,
  distancia_total_planejada
FROM `datario.transporte_rodoviario_municipal.viagem_planejada_onibus`
WHERE data >= '2023-01-06'
);

-- coleta dos dados referentes aos registros de viagens realizadas

CREATE TABLE data-science-puc.onibus_raw.viagem_onibus AS (
SELECT
 data,
 consorcio,
 tipo_dia,
 id_empresa,
 id_veiculo,
 servico,
 sentido,
 distancia_planejada,
 perc_conformidade_shape,
 perc_conformidade_registros
FROM `datario.transporte_rodoviario_municipal.viagem_onibus`
WHERE data >= '2023-01-06'
)
```
Devido à limitações do Data Fusion, as duas queries foram adicionadas no mesmo campo, para que a carga dos dois datasets pudesse ser feita de uma só vez.

![carga utilizando o data fusion](prints\carga-datario_BQ.png)

## MODELAGEM
Para o resultado e análises pretendidas, foi feita a opção por um banco de dados relacional, com um esquema *snowflake*. O esquema é então composto por seis tabelas, sendo uma tabela fato, três dimensões principais - incluindo uma dimensão tempo (tabela dia) - e duas tabelas com dados que se relacionam à dimensão consorcio-empresa.  

![esquema snowflake genMyModel](prints\esquema_modelagem.png)

- Na tabela fato (viagens_dia), cada entrada corresponde à operação de uma linha de ônibus, por empresa operadora, na data registrada. Ela contém ainda outros dados importantes como a quilometragem total registrada nessa operação e dados relacionados com outras tabelas.

- Na dimensão empresa_consórcio, cada entrada corresponde a uma combinação única empresa-consórcio, sendo assim uma chave composta. Com isso, seguindo a proposta *snowflake*, foram compostas mais duas tabelas relacionadas a essa dimensão: uma com dados relativos ao consórcio e outra com dados referentes a cada empresa. A opção por essa forma de composição dos dados é devida ao fato de que além de um consorcio ser composto por mais de uma empresa, cada empresa pode pertencer a mais de um consórcio ao mesmo tempo. Nesse sentido, a organização proposta facilida a análise dos dados por empresas, que é um dos aspectos motivadores para a composição deste banco de dados. 

- Já na dimensão linhas_onibus, temos cada linha operada, segundo seus atributos, identificadas por uma chave única, como uma espécie de surrogate (id_linha). Cada linha considerada é uma combinação única dos seguintes atributos: serviço, sentido e dia_semana. Isso ocorre porque o valor do subsídio é calculado com base na quilometragem percorrida e o número de viagens realizadas, de modo os trajetos de uma linha (serviço) de ida e volta possuem extensões distintas que podem ainda variar entre dias úteis e finais de semana.

- Por fim, a dimensão dia corresponde à dimensão temporal, facilitando a identificação das datas na tabela fato entre dias úteis, sábado ou domingo, além do identificador do mês, para facilitar a agregação nas consultas para a análise.

Apesar do esquema ter sido pensado com chaves relacionais (para garantir a integridade dos dados), o banco de dados na nuvem do Big Query, por se tratar de um banco colunar, não permite estabelecer de forma direta as relações ou habilitar restrições de integridade para as entradas de dados que estejam para além do *datatype*.

## ETL
Para a criação das tabelas pretendidas foi utilizada a ferramenta Google Data Fusion, serviço ETL disponível para Google Cloud. Nesse ponto, foi utilizada a interface Pipeline Studio, com ferramentas do tipo "point-and-click" para a execução das tarefas desejadas. Considerando as fontes de dados variadas para a composição do DW, cada tabela seguiu um fluxo específico de ETL. Também considerando algumas limitações encontradas nas ferramentas disponíveis do Pipeline Studio (especialmente a de transformação "Wrangler"), foi necessário realizar o *load* dos dados em um dataset temporário no Big Query, para depois performar as transformações finais em um segundo pipeline. 

Assim, as tabelas foram construídas conforme os procedimentos descritos a seguir:

#### *1) Tabelas empresa_consorcio, empresa e consorcio*  
As tabelas empresa_consorcio e consorcio foram compostas a partir do pipeline representado abaixo:

![pipeline tabelas empresa_consorcio e consorcio](prints\pipeline_tabelas_empresa-consorcio.png)

Com as informações contidas no dataset raw, as tabelas foram construídas a partir de operações group by, para identificar as entradas únicas e agregar os atributos desejados em cada uma. Após selecionadas as informações desejadas para a versão final de cada tabela, os dados foram sincronizados no dataset final do projeto no Big Query.

Já para a tabela empresa, foram necessárias algumas transformações adicionais, além da combinação entre fontes de dados no Big Query e dados no bucket do GCS, conforme pode ser visto no pipeline:

![pipeline tabela empresa](prints\pipeline_tabela_empresa.png)
Com os dados da tabela raw viagem_onibus, são feitas duas operações de group by, uma para determinar o número de carros por empresa  e outra para determinar o número de linhas por empresa. O nome das empresas é identificado a partir do .csv de elaboração própria originalmente armazenado no bucket, que passa apenas por uma mudança de datatype (utilizando o wrangler). As duas informações são então unidas por meio da ferramenta "Joiner", gerando então a tabela final que é sincronizada no dataset do Big Query. 

#### *2) Tabela linhas_onibus*
A construção da tabela referente as linhas de ônibus foi composta de duas etapas de transformação. Na primeira, utilizando dados diretamente da tabela raw que contém as linhas planejadas por dia pela Prefeitura. Foi então utilizada o "Wrangler" para a inclusão de uma coluna com o valor calculado para o subsidio em cada viagem e em seguida aplicada a operação de "Deduplicate", dado que na tabela original diversas linhas se repetem em mais de um dia. Foi feito ainda um "Group By" para a agregação por linha de ônibus e dia da semana, já que no dado fonte as linhas estavam separadas em trajeto de ida e volta. Ao final, foi feita a carga da tabela resultante em um dataset temporário no Big Query.
![pipeline tabela linhas](prints\pipeline_tabela_linhas_onibus.png)

Na segunda etapa, foram feitas novas transformações utilizando a ferramenta wrangler, tomando como fonte a tabela do dataset temporário criada anteriormente. Nesse ponto, foram ajustados os *datatypes* dos atributos para os tipos desejados, além de criar a coluna id_linha, como um código formado a partir da combinação entre o serviço(nº da linha) e o tipo de dia da semana(sabado, domingo ou dia útil). Foi ainda criada a coluna de "viagens_planejadas_dia" a partir do dado da quilometragem total esperada para a linha em um dia.
![wrangle2 tabela linhas](prints\wrangle_final_tabela_linhas_onibus.png)

Após essas transformações, foi feita novamente uma operação de "Deduplicate", visto que algumas linhas de onibus ainda se repetiam, guardando apenas pequenas diferenças no número de viagens planejadas por dia. Com isso, optou-se como critério para a seleção, guardar aquelas que apresentassem o maior número de viagens planejadas. Após esse procedimento, a tabela foi então sincronizada na base de dados final (dataset onibus_subsidio_rj).
![pipeline2 tabela linhas](prints\succeeded_pipeline2_linhas_onibus.png)

#### *3) Tabela viagens_dia*
No caso desta tabela que corresponde à tabela fato do esquema projetado, também foi necessário realizar as transformações em duas etapas. Na primeira, são feitos dois caminhos de transformações que são unidos ao final, tendo como fonte a tabela raw extraída do database da Prefeitura. Um dos caminhos é feito para a inclusão do valor calculado para o subsídio de cada viagem - a nova coluna foi criada no "Wragler", com a informação do valor pago por km (R$3,80) determinado em decreto municipal. Nos dois caminhos, foi feita a agregação por "Group By" das viagens por dia, unindo as informações por "Join" de forma que a tabela resultante tivesse suas entradas únicas por dia, empresa e linha. Esses dados foram então armazenados no dataset temporário para depois proceder a outras transformações.
![pipeline1 tabela viagens_dia](prints\succeeded_pipeline_tabela_viagens_dia.png)  

Na segunda etapa, foram apenas realizados ajustes nos nomes das colunas e *datatypes*, além da substituição das informações de tipo de dia da semana e serviço operado pelo id_linha, composto da mesma maneira que os códigos da tabela linhas_onibus.
![wrangle2 tabela viagens_dia](prints\wrangle_final_tabela_viagens_dia.png)

Assim, o pipeline resultante desse processamento é composto das seguintes etapas: extração dos dados da tabela de viagens no dataset temporário, transformações das colunas no "Wrangler" e carga da tabela resultante no dataset final no Big Query.
![pipeline2 tabela viagens_dia](prints\succeeded_pipeline2_viagens_dia.png)

#### *4) Tabela dia*
Para a construção da dimensão temporal do esquema projetado, foi utilizada a tabela de viagens_dia do dataset temporário, considerando que as viagens já estariam agregadas por dia-linha-empresa, reduzindo o volume de dados para o processamento em comparação à tabela raw com os dados originais.
Foram utilizadas apenas as colunas data e tipo_dia e, através da ferramenta wrangler foram acrescidas as colunas de mes e ano, partindo da separação dos dados contidos na coluna data.
![wrangle tabela dia](prints\wrangle_tabela_dia.png)

Após esse processamento, foi utilizado o "Deduplicate" para a seleção de entradas únicas de data. O pipeline resultante ficou então com quatro etapas, contando com a carga da tabela no dataset final do projeto no Big Query.
![pipeline tabela dia](prints\succeeded_pipeline_dia.png)

#### *Resultado final*
Após a execução dos procedimentos relatados, obteve-se o banco de dados composto das seis tabelas, conforme o esquema projetado. Apesar de não contar com as relações e restrições inicialmente desejadas, o banco pode ser utilizado com certa confiabilidade nas análises.
![resultado final no big query](prints\bigquery_final.png)

### LINHAGEM DOS DADOS

A linhagem dos dados registra então os caminhos dos dados contidos no dataset final, permitindo rastrear possíveis erros que venham a ser identificados e garantir a qualidade dos dados. A linhagem foi então documentada no seguinte diagrama:
![linhagem dos dados](prints\data_lineage.jpg)
Os processos documentados no diagrama podem ser de 5 tipos:

  - copia: é realizada uma cópia idêntica, integral ou parcial, sem qualquer transformação nos dados.

  - agrega: os dados são agregados em unidades de medida maiores, como por exemplo quando passa da unidade viagens no dado original para a unidade dia, agregando todas as viagens realizadas no mesmo dia.

  - agrupa: se refere às operações de *group by* em que os dados são agrupados por tum atributo específico, diferente do atributo principal da fonte original.

  - transforma: são feitas transformações como a mudança no nome de atributos, mudança de *datatype*, entre outras transformações feitas diretamente no dado.

  - adiciona coluna: nesse caso apenas é adicionada uma informação nova sem agregações ou transformações.

O registro da linhagem nesse caso foi elaborado por meio da ferramenta para diagramas "miro", uma vez que se trata de um esquema simples e, no caso, todo o processamento e uso da base está sendo realizado por um usuário único (por isso não houve também necessidade de indicação do agente nas tarefas executadas). Existem porém, outras formas de documentação, incluindo mecanismos que geram a linhagem dos dados automaticamente a partir dos processos realizados.   
No caso do registro da linhagem disponibilizado pelo próprio Big Query, o que se percebeu foi a falta de conexão com o Data Fusion - onde foram feitas as transformações, demorando ainda para carregar as atualizações. Além disso, seria necessário habilitar a ferramenta Dataplex, o que na situação em questão, começaria a estourar a cota gratuita na conta Google utilizada.

### CATÁLOGO DOS DADOS
Da mesma forma, o catálogo de dados também poderia ser construído por meio da ferramenta "Dataplex" e "Data Catalog" do Google, específica para trabalhar questões relacionadas à governança de dados. No entanto, devido à limitação já mencionada para o uso das ferramentas do Google Cloud, optou-se por documentar o catálogo de forma manual em excel gerando um conjunto de tabelas de metadados. Esse conjunto foi documentado na pasta catalogo [nesse github](https://www.xxxx).


## ANÁLISES
Uma vez estabelecido o DW e organizados os dados nas tabelas, é possível então passar à utilização dos mesmos para extrair as informações e análises desejadas. Com isso, é possível responder às perguntas iniciais que motivaram esse estudo, além de propiciar outros entendimentos sobre o sistema de ônibus.

### QUALIDADE DOS DADOS
É fundamental destacar que frequentemente os dados não se encontram disponíveis da forma adequada e clara, gerando não somente a necessidade de limpeza, mas também que seja assegurada a qualidade e confiabilidade do dado que será utilizado para apoiar a tomada de decisão. No caso do banco de dados elaborado, um aspecto relevante são os dados nulos. Especialmente no caso da tabela empresa que, por ter sido formulada a partir de fontes diversas, exigindo um cuidado maior no seu tratamento, ainda assim com um grande volume de entradas "null". O que se percebeu neste ponto foi que o dado original da Prefeitura utiliza uma codificação para o id_empresa, o que acaba gerando identificadores diferentes para um mesmo operador e prejudicando, portanto, as agregações que dependem dessa informação. Com o problema vindo da fonte, fica ainda mais difícil encontrar soluções para a qualidade desses dados.

![entradas null tabela empresa](prints\dados_nulos_empresa.png)

Essa questão é ainda complementada com a impossibilidade de incluir *constrains* no formato em que é construído o banco de dados no Big Query. Com isso, o banco pode acabar permitindo dados que violam sua característica desejada. No entanto, ao determinarmos o *datatype* de cada coluna, essa questão pode ser em parte minimizada.

Outro ponto importante em relação à qualidade, está também relacionado à impossibilidade de estabelecer relacionamentos diretos entre as tabelas, por meio de chaves primárias ou estrangeiras. No caso trabalhado, um exemplo que peca contra a qualidade dos dados nesse sentido é a definição do id_linhas, presente na tabela de viagens_dia e na tabela linhas_onibus. Apesar da intenção na modelagem ter sido de um relacionamento (foreign key e primary key respectivamente), nas etapas de transformação os códigos foram gerados a partir do mesmo procedimento porém empregado em fontes diferentes. Dessa forma, torna-se mais difícil garantir a sua correspondência e evitar divergências na informação, podendo ser encontrados id's em uma tabela (em que preocupa mais os encontrados na tabela viagens_dia) sem equivalência de dados na outra, ou até mesmo com informações conflitantes. Essa questão foi avaliada por meio da seguinte query:

```SQL
WITH nulos AS (
SELECT 
  v.id_linha IS NULL AS linhas_v,
  l.id_linha IS NULL AS linhas_l,
FROM `data-science-puc.onibus_subisidio_rj.viagens_dia` AS v
FULL OUTER JOIN `data-science-puc.onibus_subisidio_rj.linhas_onibus` AS l
ON v.id_linha = l.id_linha
)
SELECT
  "viagens_dia" AS nulos,
  COUNT(*) AS id_sem_correspondencia
FROM nulos
WHERE nulos.linhas_v
UNION ALL
SELECT
  "linhas_onibus" AS id_sem_correspondencia,
  COUNT(*) AS id_sem_correspondencia
FROM nulos
WHERE nulos.linhas_l
```
Com o resultados percebemos que há um problema de qualidade, uma vez que encontramos muitas códigos sem correspondência entre as duas tabelas:
![resultado query correspondencia ids](prints\resultado_qualidade.png)

Para garantir então a qualidade desses dados, seria necessário realizar uma etapa adicional no tratamento, garantindo a relação entre as fontes de dados ou utilizando a mesma fonte para a composição das duas tabelas.

Apesar disso, pelo fato das perguntas iniciais não focarem nas linhas operadas, é possível ainda assim prosseguir com a análise inicialmente proposta no problema.

### RESPOSTAS ÀS QUESTÕES INICIAIS
- Quantas viagens são feitas em média, em dias úteis, por cada empresa?

```SQL
SELECT 
  viagens.id_empresa,
  empresa.nome_empresa,
  AVG(viagens.total_viagens) AS media_viagens,
FROM `data-science-puc.onibus_subisidio_rj.viagens_dia` AS viagens
LEFT OUTER JOIN `data-science-puc.onibus_subisidio_rj.empresa` AS empresa 
ON viagens.id_empresa = empresa.id_empresa
JOIN `data-science-puc.onibus_subisidio_rj.dia` AS dia ON
viagens.data = dia.data
WHERE dia_semana = 'Dia Útil'
GROUP BY viagens.id_empresa,empresa.nome_empresa
ORDER BY media_viagens DESC
```
- Qual a relação média de veículos por linha operada em cada empresa?

```SQL
SELECT 
  id_empresa,
  nome_empresa,
  total_linhas_empresa/total_carros AS media_carrosporlinha
FROM `data-science-puc.onibus_subisidio_rj.empresa`
ORDER BY media_carrosporlinha DESC
```
- Qual o percentual de quilômetros rodados por cada empresa em cada consórcio nos últimos 6 meses de dados disponíveis?

```SQL
WITH data_inicial AS (
  SELECT
    DATE_SUB (MAX (data), INTERVAL 6 MONTH) AS inicio
  FROM `data-science-puc.onibus_subisidio_rj.viagens_dia`
),
km_consorcio AS (
  SELECT
    consorcio,
    SUM(total_km_percorrido) AS total_km_consorcio 
  FROM `data-science-puc.onibus_subisidio_rj.viagens_dia`
  WHERE data >= (SELECT inicio FROM data_inicial)
  GROUP BY consorcio
),
km_empresa_consorcio AS (
  SELECT 
    id_empresa,
    consorcio,
    SUM(total_km_percorrido) AS total_km_empresa_consorcio 
  FROM `data-science-puc.onibus_subisidio_rj.viagens_dia`
  WHERE data >= (SELECT inicio FROM data_inicial)
  GROUP BY id_empresa, consorcio
)
SELECT
  kec.id_empresa,
  e.nome_empresa,
  kec.consorcio,
  kec.total_km_empresa_consorcio / kc.total_km_consorcio * 100 AS percentual_km
FROM km_empresa_consorcio AS kec
JOIN `data-science-puc.onibus_subisidio_rj.empresa` AS e
  ON kec.id_empresa = e.id_empresa
JOIN km_consorcio AS kc ON kc.consorcio = kec.consorcio
```
- Qual empresa apresentou menor percentual de viagens não subsidiadas (com informidades) no último mês de dados disponíveis?
[RESPONDI DE OUTRA FORMA!!]
```SQL
WITH data_inicial AS (
  SELECT
    DATE_SUB (MAX (data), INTERVAL 1 MONTH) AS inicio
  FROM `data-science-puc.onibus_subisidio_rj.viagens_dia`
)
SELECT
  id_empresa,
  GREATEST(SUM(viagens_inconformes_shp), SUM(viagens_inconformes_gps)) AS min_inconformidades,
  LEAST(SUM(viagens_inconformes_shp) + SUM(viagens_inconformes_gps), SUM(total_viagens)) AS max_inconformidades,
FROM `data-science-puc.onibus_subisidio_rj.viagens_dia`
WHERE data >= (SELECT inicio FROM data_inicial)
GROUP BY id_empresa
ORDER BY max_inconformidades DESC
```

- Qual a regra para o subsídio tem maior percentual de violação em cada empresa?
[RESPONDER GPS_ a OUTRA REGRA É ZERO!]
```SQL
SELECT 
FROM 
```
## AUTO AVALIAÇÃO
Após a realização das análises é possível perceber alguns aspectos importantes do processo especialmente visando sua melhoria para trabalhos futuros. 

### Desafios encontrados
Primeiramente, cabe destacar que, no processo de elaboração deste material foi possível lidar com desafios reais e bastante comuns ao profissional da área de dados. A decisão por trabalhar com uma fonte de dados como o datalake da Prefeitura do Rio, permitiu compreender questões como a falta de documentação precisa dos dados e a ausência de dados em alguns casos. Isso porque, algumas das informações trazidas para o dataset precisaram ser obtidas por meio de ampla pesquisa na internet - a exemplo do valor utilizado como base de cálculo do subsídio - além do próprio conhecimento prévio já acumulado sobre o assunto.

Além disso foram encontradas algumas limitações na ferramenta utilizada para os procedimentos de ETL, que levaram à construção de mecanismos adicionais - como o dataset temporário - para a manipulação dos dados. No caso em questão, uma vez que a fonte de dados já se encontrava no Big Query, o processo de ETL poderia se dar de forma mais eficiente através do uso de queries em SQL ou da própria ferramenta de criação de tabelas do Big Query. Ainda assim, como método didático foi importante o conhecimento das possibilidades oferecidas pelo Data Fusion, que permite realizar transformações ainda que o usuário possua pouco conhecimento de programação.

É necessário ainda destacar que, não foi possível garantir agumas das condições de integridades dos dados utilizando o Big Query como nuvem para o armazenamento. Assim, apesar da modelagem ter previsto restrições, chaves e outras condicionantes, não haviam ferramentas para que elas fizessem parte do banco de dados final. Com isso, uma possível ingestão de dados futura no banco estabelecido, poderia permitir eventuais problemas na qualidade dos dados do banco. Nesse ponto, também avaliou-se que para a utilização desse tipo de nuvem, seria talvez mais adequado um esquema em tabela única, por ser um banco que utiliza o sistema colunar, em que não impactaria tanto uma grande quantidade de atributos ainda que a tabela tivesse muitos dados.

É também importante mencionar a questão das limitações de custos financeiros implicadas no uso da nuvem. Os custos para a utilização das ferramentas é bastante alto e a fração disponibilizada gratuitamente restringe enormemente as suas possibilidades de uso, inclusive a exploração das melhores soluções disponíveis, como por exemplo as ferramentas de auxílio à governança de dados. Ainda assim, foi possível executar o trabalho e responder ès perguntas iniciais com custos relativamente baixos, porém fazendo restrições à quantidade de dados para tal.

### Perspectivas para trabalhos futuros
Uma vez compreedidas todas as etapas do processo, podemos então ver pontos de melhoria para outros trabalhos. Considerando a temática específica do sistema de ônibus, caberia primeiramente buscar outras fontes confiáveis para estabelecer a relação entre o identificador das empresas utilizadas na base da prefeitura e os respectivos nomes, para que assim fosse possível ter respostas mais precisas e completas sobre o problema apresentado. Além disso, a tabela referente à dimensão tempo (tabela dia) ajudou pouco na solução dos problemas, uma vez que em SQL é possível realizar operações em cima de datas. Assim, consideraria que a inculsão dos dados da mesma em na tabela fato poderia tornar o esquema mais simples e não implicaria necessariamente em maior custo computacional. 

Para as perguntas elaboradas, a utilização da tabela "linhas_onibus" resultou desnecessária, já que não foram feitas análises por meio dela. Porém, a presença da tabela em um dataset como esse permitiria responder outras perguntas de possível interesse, como quantas empresas operam cada linha, quais são as linhas que recebem maior valor de subsídio ou quais as penalidades do subsídio costumam ser mais frequentes em cada linha.

Para a melhora desse trabalho, seria interessante também agregar ferramentas de visualização de dados, para a elaboração de dashboards e gráficos das análises. Com isso, a compreensão desejada a partir do banco de dados e a sua comunicação poderia se dar de forma mais imediata e eficaz.

Por fim, como forma de incremento deste trabalho, poderiam ser também adicionados outros dados de interesse, para análises ainda mais complexas. A exemplo disso, a análise das viagens e do respectivo subsídio poderia ainda ganhar uma componente espacial, incluindo dados sobre os bairros atendidos por cada consórcio ou questões relacionadas à localidade das linhas operadas por cada empresa.
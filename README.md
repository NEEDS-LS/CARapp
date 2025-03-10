
<!-- README.md is generated from README.Rmd. Please edit that file -->

# restauraRapp

<!-- badges: start -->
<!-- badges: end -->

Este documento descreve as funcionalidades do pacote “restauRapp”. O
Objetivo do pacote é auxiliar na delimitação dos passivos ambientais nas
áreas de preservação permanentes (APPs) hídricas de propriedades rurais
no território brasileiro.

Com a promulgação da Lei de Proteção da Vegetação Nativa (LPVN - [Lei
12.651, de 25 de maio de
2012](http://www.planalto.gov.br/ccivil_03/_ato2011-2014/2012/lei/l12651.htm),
ou Novo Código Florestal) em 2012, a delimitação das Áreas de
Preservação Permanentes (APPs) passíveis de serem restauradas foi
alterada, e tornou-se dependente do tamanho das propriedades, baseado no
número de módulos fiscais. Consequentemente, informações referentes ao
tamanho do módulo fiscal, que varia de município para município, e o
tamanho da propriedade, que pode ser obtido através do
[CAR](https://www.car.gov.br/) (Cadastro Ambiental Rural) são
necessárias para a correta delimitação das áreas de passivo ambiental.

Este pacote busca auxiliar exatamente nessa tarefa, particularmente
focando no cálculo das APPs de cursos d’água de acordo com o tamanho das
propriedades cadastradas no *Sistema Nacional de Cadastro Ambiental
Rural* ([SICAR](https://www.car.gov.br/publico/imoveis/index)).

Para a criação das áreas a serem restauradas é usado a hidrografia
disponibilizada na base de dados da Fundação Brasileira para o
Desenvolvimento Sustentável ([FBDS](https://www.fbds.org.br/), os dados
podem ser encontrados no link <http://geo.fbds.org.br/>.

As informações cartográficas sobre o uso do solo podem ser de diversas
fontes. O padrão é o dado também disponível na base de dados da
[FBDS](https://www.fbds.org.br/), contudo outras fontes de dados como o
MapBiomas, ou qualquer outro mapeamento, também podem ser utilizados
para fornecer estas informações.

Inicialmente precisamos baixar o pacote *restauRapp* e iremos fazer isso
do [repositório do NEEDS](https://github.com/NEEDS-LS) no gitHub. Caso
você não tenha o pacote devtools, comece baixando ele.

## Instalação

Você pode instalar a versão de desenvolvedor do pacote restauraRapp do
[GitHub](https://github.com/) com:

``` r
# install.packages("devtools")
devtools::install_github("NEEDS-LS/restauraRapp")
```

## Primeiros passos

A primeira coisa a se fazer após a intalação é carregar o pacote para o
R.

``` r
library(restauraRapp)
#> Warning: package 'terra' was built under R version 4.3.2
```

Para obter os dados de hidrografia e uso do solo padrões para o pacote
(FBDS) basta executar a função *resapp_fbds_dados()*. Esta função recebe
a sigla do estado a qual o município foco esta localizado e o nome do
município em caixa alta, separado por “\_” quando necessário, como
mostrado no exemplo a seguir. É recomendado a criação de um projeto
(Arquivos/Novo projeto no menu do RProject) antes da execução das
funções aqui mostradas, pois muitas delas criam pastas para armazenar os
resultados, mantendo o ambiente organizado. Caso as funções sejam
executadas sem a existência de um projeto, as pastas e arquivos serão
salvas no diretório de trabalho do R, que pode ser obtido ao executar a
função base do R *getwd()*.

``` r

#exemplo de como usar a função resapp_fbds_dados(), para baixar um município do seu interesse basta trocar o estado e o nome, vale ressaltar que ambos precisam estar em MAIUSCULO.

resapp_fbds_dados("SP","CAMPINA_DO_MONTE_ALEGRE")

#caso tenha executado a função sem ter criado um projeto, executar a linha abaixo e veja em que 
#diretório seus dados foram salvos

getwd()

#Outro exemplo 
#resapp_fbds_dados("SP","BURI")
```

Esta função irá criar um diretório dentro do seu projeto para salvar as
informações, chamada *./dados/FBDS\_(nome do município)*. Todo novo
município será salvo na mesma pasta *data* e uma nova pasta *FBDS\_*
será criada.

Os dados do CAR estão disponiveis na plataforma do
[SICAR](https://car.gov.br/publico/imoveis/index) e o download é feito
por estado, selecionando a opçãp *perímetros dos imóveis*. O Download
destes dados devem ser feitos manualmente devido a presença de um
“captcha” que impede o acesso ao link de download diretamente.

É necessário inicialmente tratarmos algumas práticas, como o processo
para abrir os dados e o sistema de referência de coordenadas (do inglês
CRS - Coordinate Reference System). Para abrir os dados, deve-se
utilizar a função *st_read()* do pacote *sf*, preenchendo o diretório
onde o arquivo está salvo e em “layer” o nome do arquivo. Já para o CRS,
sempre utilizar aquele que melhor se enquadre em sua região desde que o
mesmo esteja em UTM (unidade em metros) e essa conversão deve ser feita
para todas informações cartográficas utilizadas. Para isso podemos
utilizar a função especifica proveniente do pacote sf, como mostrado no
exemplo abaixo.

``` r
#carrega o arquivo de usos e cobertura do solo na memória do R
USO<-st_read("seu diretório de dados/data/FBDS_SEU_MUNICIPIO", layer="ESTADO_CODIGODOMUNICIPIO_USO")
#carrega o arquivo de massas d'água na memória do R
MDA<-st_read("seu diretório de dados/data/FBDS_SEU_MUNICIPIO", layer="ESTADO_CODIGODOMUNICIPIO_MASSAS_DAGUA")
#carrega o arquivo de rios de margem simples (< 10 metros) na memória do R
RMS<-st_read("seu diretório de dados/data/FBDS_SEU_MUNICIPIO", layer="ESTADO_CODIGODOMUNICIPIO_RIOS_SIMPLES")
#carrega o arquivo de rios de margem dupla (> 10 metros) na memória do R
RMD<-st_read("seu diretório de dados/data/FBDS_SEU_MUNICIPIO", layer="ESTADO_CODIGODOMUNICIPIO_RIOS_DUPLOS")
#carrega o arquivo de nascentes na memória do R
NAS<-st_read("seu diretório de dados/data/FBDS_SEU_MUNICIPIO", layer="ESTADO_CODIGODOMUNICIPIO_NASCENTES")
#carrega o arquivo do CAR na memória do R
CAR<-st_read("seu diretório de dados", layer="AREA_IMOVEL_1") #AREA_IMOVEL_1 é o nome do arquivo baixado do SICAR
```

Algumas vezes os arquivos espaciais, principalmente aqueles formados por
polígonos, como o uso do solo e as hidrografias contendo os rios de
margem dupla e massas d’água, podem conter algum erros, como vértices
duplicados ou intercessão de linhas do polígono. Uma forma de solucionar
isso é criar um buffer nesses objetos de comprimento “0” ou executar a
função “st_make_valid” do pacote “sf”. Normalmente o buffer funciona
melhor, contudo a problemas que nem permitem a execução do buffer, neste
caso a solução será executar o “st_make_valid”. Esses métodos serão
utilizados para os casos que forem encontrados geometrias inválidas,
como por exemplo:

``` r
USO<-st_buffer(USO, 0)#exemplo de correção com o buffer de comprimento "0"

USO<-st_make_valid(USO)#exemplo de correção com a função do pacote "sf"
```

## Criando o buffer

A partir daqui executaremos as funções deste pacote. As funções podem
ser executadas tanto usando os dados que você baixou como com os dados
disponíveis dentro do pacote para você se familiarizar com as funções.
Agora iremos verificar quais conjuntos de dados tempos disponíveis.

``` r
data(package = "restauraRapp") 
```

Caso opte pelos dados dinponiveis no pacote, iremos usar como exemplo os
dados do município de Campina do Monte Alegre, no estado de São Paulo.
Esse município esta localizado próximo ao campus [Lagoa do
Sino](https://www.lagoadosino.ufscar.br/) da Universidade Federal de São
Carlos ([UFSCar](https://www2.ufscar.br/)) onde o
[NEEDS](https://www.needs.ufscar.br/) fica locaizado.

``` r
data("CMA")
```

Vamos dar inicio ao teste das funções, começando pela função
*resapp_car_class()*. O parâmetro nesta função é apenas o objeto na qual
as informações do CAR estão mantidos e o retorno é uma lista com os
tamanhos separados de acordo com o número de módulos fiscais divididos
nos grupos: micro (\< 1 módulo fiscal), pequenas entre 1 e 2 módulos
fiscais, pequenas entre 2 e 4 módulos fiscais e grandes (\> 4 módulos
fiscais). Como os dados do CAR são disponibilizados na escala estadual,
o município precisa ser selecionado antes de ser usado nesta e nas
demais funções (veja o exemplo). O nome do município deve ser colocado
sem ascentos.

``` r

#usando o exemplo de Campina do Monte Alegre, Não executar
#CMA_CAR<-SP_CAR[SP_CAR$municipio == "Campina do Monte Alegre",]

#usando os dos carregados nos *Primeiros Passos*
MUN_CAR<-CAR[CAR$municipio == "Nome do Municipio",]

#execução da função usando o exemplo de Campina do Monte Alegre
propriedades<-resapp_car_class(CMA_CAR)

#caso esteja usando outro município carregado nos *Primeiros passos* basta trocar o nome da variavel
propriedades<-resapp_car_class(MUN_CAR)

micro<-propriedades[[1]]
#peq12<-propriedades[[2]]
#peq24<-propriedades[[3]]
#grande<-propriedades[[4]]


micro
```

A função de conversão de CRS *st_transform()* (abaixo), devem ser
executadas para carregar e alterar o CRS de todos os arquivos
utilizados. No caso de Campina do Monte Alegre, usaremos o EPSG 31982,
referente ao SIRGAS 2000/ UTM zone 22s. Não esqueça dessa etapa caso
esteja utilizando dados baixados para um município de sua escolha, se
não erro irá aparece na sua tela: “Error in
geos_op2_geom(”intersection”, x, y, …) : st_crs(x) == st_crs(y) is not
TRUE”

``` r
CMA_USO<-st_transform(CMA_USO, 31982)
CMA_MDA<-st_transform(CMA_MDA, 31982) 
CMA_RMS<-st_transform(CMA_RMS, 31982) 
CMA_RMD<-st_transform(CMA_RMD, 31982) 
CMA_NAS<-st_transform(CMA_NAS, 31982)
CMA_CAR<-st_transform(CMA_CAR, 31982)

#exemplo de como alterar o CRS, atenção para mudar o CRS de todos os arquivos para evitar inconsistências.
USO<-st_transform(USO, numero do CRS)
MDA<-st_transform(MDA, numero do CRS) 
RMS<-st_transform(RMS, numero do CRS) 
RMD<-st_transform(RMD, numero do CRS) 
NAS<-st_transform(NAS, numero do CRS)
CAR<-st_transform(CAR, numero do CRS)
```

Agora vamos executar a função *resapp_app_buffer()* que vai criar os
buffers e recortar o uso de solo dentro das áreas definidas para
restauração para cada classe de propriedades. Essa função demanda um
poder de processamento considerável e pode demorar, por esse motivo
atualmente ela se encontra com um parâmetro para ser executada em
partes, separada por classe de tamanho, e outro para execução completa
das cinco diferentes classes.

Também se encontra disponivel dois cenários para avaliação das áreas sem
CAR, considerando que todo o território é ocupado por propriedades
rurais. No primeiro cenário tratamos essas áreas sem CAR como micro
propriedades e no segundo consideramos toda a área como grande
propriedade, isso representa os valores mínimos e máximos que podem ser
restaurados nessas áreas.

Caso esteja usando os dados de um município de sua escolha apenas troque
o nome das variaveis (que começam com “CMA\_”) e coloque os nomes das
variaveis utilizadas (se estiver seguindo este documento, apenas remova
o “CMA\_”).

``` r

#função para execução de todas as classes de tamanho segundo o CAR
REST_prop<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "tudo")

#função para a execução por partes das classes de tamanho segundo o CAR
REST_micro<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "micro")
REST_peq12<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "peq1")
REST_peq24<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "peq2")
REST_media<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "media")
REST_grand<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "grande")

#função para execução dos cenários referentes as áreas sem CAR registrado
#tudo sendo micro propriedade
REST_cena1<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "out1")

#tudo sendo grande propriedade
REST_cena2<-resapp_app_buffer(CMA_MDA, CMA_RMS, CMA_RMD, 
                              CMA_NAS, CMA_CAR, CMA_USO, tipo = "out2")
```

Aqui mostramos como unir todos os objetos e um exemplo para salvar as
informações. Caso aconteça algum problema com uso de memória pelo R é
interessante salvar cada classe de tamanho em separado e posteriormente
junta-lo.

``` r
mapa_geral<-rbind(REST_micro, REST_peq12, REST_peq24, REST_media, REST_grand)
st_write(mapa_geral, dsn="./sua_pasta_de_dados", "CAR_APP_CMA_SP", 
         driver="ESRI Shapefile")
```

Caso tenha executado a função como “tudo”, ou seja, todas as classes de
uma vez, basta salvar o objeto resultante.

``` r
st_write(REST_prop, dsn="./sua_pasta_de_dados", "CAR_APP_CMA_SP", 
         driver="ESRI Shapefile")
```

E, por fim, salvar os dois cenários criados

``` r
st_write(REST_cena1, dsn="./sua_pasta_de_dados", "OUT_M_APP_CMA_SP", 
         driver="ESRI Shapefile")

st_write(REST_cena2, dsn="./sua_pasta_de_dados", "OUT_G_APP_CMA_SP", 
         driver="ESRI Shapefile")
```

Como resultado temos o uso do solo dentro do buffer para restauração e
podemos calcular a quantidade de área com vegetação nativa e o que
precisa ser recuperado. Deste modo apresentamos a função
*resapp_app_info()*, está função é responsavel por retornar três
diferentes tipos de resultados: I) as áreas que, separadas por classe de
tamanho que precisam ser restauradas ou que se encontram preservadas com
informações cartográficas; II) Informações das áreas a serem restauradas
ou que se encontram preservadas por classe de tamanho na forma de tabela
de dados para impressão; III) Uma tabela de dados contendo as
informações do que deve ser resutarado ou se encontra preservado por
propriedade.

``` r
#Para executar o tipo "df" é necessário todos os resultados da função gCARapp()
  dados.CARapp<-resapp_app_info(CMA_APP,CMA_OUTM,CMA_OUTG,tipo="df")

# A opção "all" é executada apenas para os resultados das análises das áreas que possuem CAR.
  poligonos.CARapp<-resapp_app_info(CMA_APP, tipo="all")

# Por fim, a opção "prop" leva em consideração apenas os resultados das análises das áreas que possuem CAR.
  propriedade.CARapp<-resapp_app_info(CMA_APP, CMA_CAR, tipo="prop")
```

Aqui, por exemplo, está a tabela de dados das APPs por classe de
tamanho.

``` r
knitr::kable(dados.CARapp)
```

Por fim, vamos dar uma olhada em algumas informações sobre o CAR que
podem ser obtidas usando a função *resapp_car_info()*.

``` r
df.CAR<-resapp_car_info(CMA_CAR)
```

``` r
knitr::kable(df.CAR)
```

Para exportar em KML é utilizada a função *resapp_app_kml()*, ela
retornara os poligonos em um formato apto para visualização e validação
no Google Earth. Lembre-se de trocar a variável, o nome e o estado caso
esteja trabalhando com outro município.

``` r
resapp_app_kml(CMA_APP, "Campina do Monte Alegre", "SP")
```

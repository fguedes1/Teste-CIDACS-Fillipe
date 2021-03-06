Processo Seletivo CIDACS
================
Fillipe Guedes Soares
2022-07-07

Pacotes Utilizados:

``` r
library(tidyverse)
library(stringr)
library(arrow)
library(qwraps2)
library(fastDummies)
library(kableExtra)
library(fixest)
```

# Baixando os dados e Importando para o R

Os dados utilizados neste exercício foram obtidos a partir do site:
<https://opendatasus.saude.gov.br/>

Para fins de realização deste exercício, foram usados os anos de 2019 e
2020. Depois de Baixados para um diretório do computador, vou carregar
as bases para os anos de 2020 e 2019 no R:

``` r
Mortalidade_Geral_2020 <- read.csv("Mortalidade_Geral_2020.csv", header = T, sep = ';')
Mortalidade_Geral_2019 <- read.csv("Mortalidade_Geral_2019.csv", header = T, sep = ';')
```

Antes de fazer a integração das bases, será criada uma variável chamada
Ano para melhor identificar cada uma das mesmas posteriormente

``` r
Mortalidade_Geral_2020 = Mortalidade_Geral_2020 %>%
  mutate(Ano = 2020)

Mortalidade_Geral_2019 = Mortalidade_Geral_2019 %>%
  mutate(Ano = 2019)
```

Agora, vou compatibilizar (juntar) as duas bases por linhas e, ao mesmo
tempo, excluir as bases dos anos individuais visando ficar com o
ambiente global de trabalho mais bem organizado

``` r
Mortalidade_Geral_2020_2019 = rbind(Mortalidade_Geral_2020, Mortalidade_Geral_2019)
rm(Mortalidade_Geral_2020)
rm(Mortalidade_Geral_2019)
```

Aqui, estarei observando especificamente a questão de óbitos por motivos
nutricionais. Sendo assim, abaixo serão filtrados os códigos de acordo
com a CID-10: Desnutrição (E40-E46), Deficiência de Vitamina A (E50),
Outras deficiências vitamínicas (E51-E56), Sequelas da Desnutrição
(E64).

``` r
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(Nutricao = ifelse(grepl('E40|E41|E42|E43|E44|E45|E46|E50|E51|E52|E53|E54|E55|E56|E64', CAUSABAS), 1, 0))

# Selecionar apenas as variáveis de interesse:

Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  select(DTNASC, DTOBITO, ESC2010, LOCOCOR, RACACOR, SEXO, CODMUNRES, Ano, Nutricao)
```

# Tratamento dos Dados

Antes de realizar qualquer estatística descritiva a respeito dos dados
do SIM, será realizada uma sequência de manipulações nas seguintes
variáveis a fim de tornar mais fácil a apresentação dos resultados
posteriores:

-   Idade (em anos)
-   Escolaridade
-   Local
-   Cor/Raça
-   Sexo
-   Região

``` r
# Para calcular a idade em anos:

Mortalidade_Geral_2020_2019$DTNASC = str_pad(Mortalidade_Geral_2020_2019$DTNASC, 8, pad = "0")
Mortalidade_Geral_2020_2019$DTOBITO = str_pad(Mortalidade_Geral_2020_2019$DTOBITO, 8, pad = "0")
Mortalidade_Geral_2020_2019$DTNASC = sub("(.{2})(.*)", "\\1-\\2", Mortalidade_Geral_2020_2019$DTNASC)
Mortalidade_Geral_2020_2019$DTNASC = sub("(.{5})(.*)", "\\1-\\2", Mortalidade_Geral_2020_2019$DTNASC)
Mortalidade_Geral_2020_2019$DTOBITO = sub("(.{2})(.*)", "\\1-\\2", Mortalidade_Geral_2020_2019$DTOBITO)
Mortalidade_Geral_2020_2019$DTOBITO = sub("(.{5})(.*)", "\\1-\\2", Mortalidade_Geral_2020_2019$DTOBITO)

# Transformando as strings em datas

Mortalidade_Geral_2020_2019$DTNASC = as.Date(Mortalidade_Geral_2020_2019$DTNASC, format = "%d-%m-%Y")
Mortalidade_Geral_2020_2019$DTOBITO = as.Date(Mortalidade_Geral_2020_2019$DTOBITO, format = "%d-%m-%Y")

Mortalidade_Geral_2020_2019$diff_time = difftime(Mortalidade_Geral_2020_2019$DTOBITO, Mortalidade_Geral_2020_2019$DTNASC, units = "days")

Mortalidade_Geral_2020_2019$diff_time = as.numeric(Mortalidade_Geral_2020_2019$diff_time)

Mortalidade_Geral_2020_2019$diff_time = Mortalidade_Geral_2020_2019$diff_time/365

Mortalidade_Geral_2020_2019$IDADE_NOVO = floor(Mortalidade_Geral_2020_2019$diff_time)

Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  select(-DTNASC, -DTOBITO)
```

``` r
# Colocando os labels na variável de escolaridade
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(ESC2010_NOVO = ifelse(ESC2010==0, 'Sem Escolaridade', 
                               ifelse(ESC2010==1, 'Fundamental I (1ª a 4ª série)',
                                      ifelse(ESC2010 == 2, 'Fundamental II (5ª a 8ª série)',
                                             ifelse(ESC2010==3, 'Ensino Médio',
                                                    ifelse(ESC2010==4, 'Ensino Superior Incompleto',
                                                           ifelse(ESC2010==5, 'Ensino Superior Completo',
                                                                  ifelse(ESC2010==9, 'Ignorado',
                                                                         
                                                                         NA))))))))
```

``` r
# Local de
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(LOCOCOR_NEW = ifelse(LOCOCOR==1, 'Hospital', 
                              ifelse(LOCOCOR==2, 'Outros Estabelecimentos de Saúde',
                                     ifelse(LOCOCOR==3, 'Domicílio', 
                                            ifelse(LOCOCOR==4, 'Via Pública', 
                                                   ifelse(LOCOCOR==5, 'Outros', 
                                                          ifelse(LOCOCOR==6, 'Aldeia Indígena', 
                                                                 ifelse(LOCOCOR==9, 'Ignorado', NA))))))))
```

``` r
# Cor/raça
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(RACACOR_NOVO = ifelse(RACACOR == 1, 'Branca', 
                               ifelse(RACACOR == 2, 'Preta', 
                                      ifelse(RACACOR == 3, 'Amarela', 
                                             ifelse(RACACOR == 4, 'Parda', 
                                                    ifelse(RACACOR == 5, 'Indígena', NA))))))
```

``` r
# Sexo
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(SEXO_NOVO = ifelse(SEXO==1, 'Masculino', 
                            ifelse(SEXO==2, 'Feminino', NA)))
```

``` r
# Criando a variável UF
Mortalidade_Geral_2020_2019$UF = substr(Mortalidade_Geral_2020_2019$CODMUNRES, 1,2)

# CRIANDO A VARIÁVEL DE REGIÃO
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(REGIAO = ifelse(UF %in% c('12', '16', '13', '15', '11', '14', '17'), 'Norte',
                         ifelse(UF %in% c('27', '29', '23', '21', '25', '26', '22', '24', '28'), 'Nordeste',
                                ifelse(UF %in% c('33', '35', '31', '32'), 'Sudeste',
                                       ifelse(UF %in% c('52', '51', '50', '53'), 'Centro-Oeste',
                                              ifelse(UF %in% c('43', '41', '42'), 'Sul', NA
                                              ))))))
```

``` r
#Por fim, criando faixas de idade
Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  mutate(FAIXA_ETARIA = ifelse(IDADE_NOVO <1, 'Menor que 1 ano',
                               ifelse(IDADE_NOVO >=1 & IDADE_NOVO <=4, '1 a 4 anos',
                                      ifelse(IDADE_NOVO >=5 & IDADE_NOVO<=9, '5 a 9 anos',
                                             ifelse(IDADE_NOVO >=10 & IDADE_NOVO<=14, '10 a 14 anos',
                                                    ifelse(IDADE_NOVO >=15 & IDADE_NOVO<=19, '15 a 19 anos',
                                                           ifelse(IDADE_NOVO >=20 & IDADE_NOVO <=39, '20 a 39 anos',
                                                                  ifelse(IDADE_NOVO >=40 & IDADE_NOVO < 60, '40 A 60 anos',
                                                                         ifelse(IDADE_NOVO>=60, '60 anos ou mais',
                                                                         NA)))))))))
```

Criando dummies das variáveis categóricas

``` r
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'ESC2010_NOVO', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'REGIAO', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'SEXO_NOVO', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'LOCOCOR_NEW', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'FAIXA_ETARIA', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'RACACOR_NOVO', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'Ano', ignore_na = T)
Mortalidade_Geral_2020_2019 = dummy_cols(Mortalidade_Geral_2020_2019, select_columns = 'Nutricao', ignore_na = T)

# Limpando a base

Mortalidade_Geral_2020_2019 = Mortalidade_Geral_2020_2019 %>%
  select(17:54)
```

# Estatísticas Descritivas

Tendo realizada a limpeza de dados da anteriormente, nesta seção
busca-se descrever estatisticamente algumas características importantes
de óbitos registrados por questões nutricionais.

``` r
# Filtrando apenas o grupo dos obitos por causas relacionadas a desnutrição
desnutricao = Mortalidade_Geral_2020_2019 %>%
  filter(Nutricao_1==1)
```

``` r
# Médias do Período

# 2020
descritivas_2020 = desnutricao %>%
  filter(Ano_2020==1) %>%
  summarise_all(mean,na.rm=T)

nomes =  as.data.frame(colnames(descritivas_2020))

descritivas_2020 = as.data.frame(as.matrix(t(descritivas_2020)))

descritivas_2020 = rownames_to_column(descritivas_2020, 'Variável')

names(descritivas_2020)[2] = '2020'

descritivas_2020$Variável = gsub(".*_", "", descritivas_2020$Variável)

descritivas_2020$`2020` = round(descritivas_2020$`2020` * 100,2)

# 2019
descritivas_2019 = desnutricao %>%
  filter(Ano_2019==1) %>%
  summarise_all(mean,na.rm=T)


nomes =  as.data.frame(colnames(descritivas_2019))

descritivas_2019 = as.data.frame(as.matrix(t(descritivas_2019)))

descritivas_2019 = rownames_to_column(descritivas_2019, 'Variável')

names(descritivas_2019)[2] = '2019'

descritivas_2019$Variável = gsub(".*_", "", descritivas_2019$Variável)

descritivas_2019$`2019` = round(descritivas_2019$`2019` * 100,2)


descritivas = cbind(descritivas_2019, descritivas_2020[,"2020"])
names(descritivas)[3] = '2020'
descritivas$Diff = descritivas$`2020` - descritivas$`2019`
descritivas = descritivas[-c(35:38),]
```

Para montar a Tabela de Estatística Descritiva, será utilizado o comando
abaixo:

``` r
kbl( descritivas
     , booktabs = T, align = 'c',  
     caption = "Estatísticas Descritivas - Perentuais")  %>%
  pack_rows("Escolaridade", 1, 7, latex_gap_space = "2em") %>%
  pack_rows("Região", 8, 12, latex_gap_space = "2em") %>%
  pack_rows("Sexo", 13, 14, latex_gap_space = "2em") %>%
  pack_rows("Local", 15, 21, latex_gap_space = "2em") %>%
  pack_rows("Faixa-Etária", 22, 29, latex_gap_space = "2em") %>%
  pack_rows("Cor/Raça", 30, 33, latex_gap_space = "2em") 
```

<table>
<caption>
Estatísticas Descritivas - Perentuais
</caption>
<thead>
<tr>
<th style="text-align:center;">
Variável
</th>
<th style="text-align:center;">
2019
</th>
<th style="text-align:center;">
2020
</th>
<th style="text-align:center;">
Diff
</th>
</tr>
</thead>
<tbody>
<tr grouplength="7">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Escolaridade</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Ensino Médio
</td>
<td style="text-align:center;">
6.14
</td>
<td style="text-align:center;">
7.39
</td>
<td style="text-align:center;">
1.25
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Ensino Superior Completo
</td>
<td style="text-align:center;">
1.66
</td>
<td style="text-align:center;">
1.88
</td>
<td style="text-align:center;">
0.22
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Ensino Superior Incompleto
</td>
<td style="text-align:center;">
0.22
</td>
<td style="text-align:center;">
0.25
</td>
<td style="text-align:center;">
0.03
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Fundamental I (1ª a 4ª série)
</td>
<td style="text-align:center;">
33.81
</td>
<td style="text-align:center;">
32.50
</td>
<td style="text-align:center;">
-1.31
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Fundamental II (5ª a 8ª série)
</td>
<td style="text-align:center;">
8.76
</td>
<td style="text-align:center;">
8.68
</td>
<td style="text-align:center;">
-0.08
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Ignorado
</td>
<td style="text-align:center;">
12.79
</td>
<td style="text-align:center;">
11.27
</td>
<td style="text-align:center;">
-1.52
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Sem Escolaridade
</td>
<td style="text-align:center;">
36.62
</td>
<td style="text-align:center;">
38.03
</td>
<td style="text-align:center;">
1.41
</td>
</tr>
<tr grouplength="5">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Região</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Centro-Oeste
</td>
<td style="text-align:center;">
7.02
</td>
<td style="text-align:center;">
7.16
</td>
<td style="text-align:center;">
0.14
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Nordeste
</td>
<td style="text-align:center;">
34.35
</td>
<td style="text-align:center;">
37.74
</td>
<td style="text-align:center;">
3.39
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Norte
</td>
<td style="text-align:center;">
9.43
</td>
<td style="text-align:center;">
9.94
</td>
<td style="text-align:center;">
0.51
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Sudeste
</td>
<td style="text-align:center;">
38.57
</td>
<td style="text-align:center;">
34.31
</td>
<td style="text-align:center;">
-4.26
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Sul
</td>
<td style="text-align:center;">
10.62
</td>
<td style="text-align:center;">
10.84
</td>
<td style="text-align:center;">
0.22
</td>
</tr>
<tr grouplength="2">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Sexo</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Feminino
</td>
<td style="text-align:center;">
47.54
</td>
<td style="text-align:center;">
48.03
</td>
<td style="text-align:center;">
0.49
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Masculino
</td>
<td style="text-align:center;">
52.46
</td>
<td style="text-align:center;">
51.97
</td>
<td style="text-align:center;">
-0.49
</td>
</tr>
<tr grouplength="7">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Local</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Aldeia Indígena
</td>
<td style="text-align:center;">
0.06
</td>
<td style="text-align:center;">
0.02
</td>
<td style="text-align:center;">
-0.04
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Domicílio
</td>
<td style="text-align:center;">
29.00
</td>
<td style="text-align:center;">
37.03
</td>
<td style="text-align:center;">
8.03
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Hospital
</td>
<td style="text-align:center;">
60.86
</td>
<td style="text-align:center;">
53.43
</td>
<td style="text-align:center;">
-7.43
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Ignorado
</td>
<td style="text-align:center;">
0.02
</td>
<td style="text-align:center;">
0.06
</td>
<td style="text-align:center;">
0.04
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Outros
</td>
<td style="text-align:center;">
2.05
</td>
<td style="text-align:center;">
1.86
</td>
<td style="text-align:center;">
-0.19
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Outros Estabelecimentos de Saúde
</td>
<td style="text-align:center;">
7.59
</td>
<td style="text-align:center;">
7.27
</td>
<td style="text-align:center;">
-0.32
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Via Pública
</td>
<td style="text-align:center;">
0.42
</td>
<td style="text-align:center;">
0.33
</td>
<td style="text-align:center;">
-0.09
</td>
</tr>
<tr grouplength="8">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Faixa-Etária</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
1 a 4 anos
</td>
<td style="text-align:center;">
1.34
</td>
<td style="text-align:center;">
0.94
</td>
<td style="text-align:center;">
-0.40
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
5 a 9 anos
</td>
<td style="text-align:center;">
0.55
</td>
<td style="text-align:center;">
0.38
</td>
<td style="text-align:center;">
-0.17
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
10 a 14 anos
</td>
<td style="text-align:center;">
0.48
</td>
<td style="text-align:center;">
0.48
</td>
<td style="text-align:center;">
0.00
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
15 a 19 anos
</td>
<td style="text-align:center;">
0.53
</td>
<td style="text-align:center;">
0.63
</td>
<td style="text-align:center;">
0.10
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
20 a 39 anos
</td>
<td style="text-align:center;">
2.78
</td>
<td style="text-align:center;">
2.64
</td>
<td style="text-align:center;">
-0.14
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
40 A 60 anos
</td>
<td style="text-align:center;">
8.49
</td>
<td style="text-align:center;">
8.76
</td>
<td style="text-align:center;">
0.27
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
60 anos ou mais
</td>
<td style="text-align:center;">
83.28
</td>
<td style="text-align:center;">
83.68
</td>
<td style="text-align:center;">
0.40
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Menor que 1 ano
</td>
<td style="text-align:center;">
2.54
</td>
<td style="text-align:center;">
2.49
</td>
<td style="text-align:center;">
-0.05
</td>
</tr>
<tr grouplength="4">
<td colspan="4" style="border-bottom: 1px solid;">
<strong>Cor/Raça</strong>
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Amarela
</td>
<td style="text-align:center;">
0.92
</td>
<td style="text-align:center;">
0.90
</td>
<td style="text-align:center;">
-0.02
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Branca
</td>
<td style="text-align:center;">
45.69
</td>
<td style="text-align:center;">
43.21
</td>
<td style="text-align:center;">
-2.48
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Indígena
</td>
<td style="text-align:center;">
1.81
</td>
<td style="text-align:center;">
1.76
</td>
<td style="text-align:center;">
-0.05
</td>
</tr>
<tr>
<td style="text-align:center;padding-left: 2em;" indentlevel="1">
Parda
</td>
<td style="text-align:center;">
42.50
</td>
<td style="text-align:center;">
44.74
</td>
<td style="text-align:center;">
2.24
</td>
</tr>
<tr>
<td style="text-align:center;">
Preta
</td>
<td style="text-align:center;">
9.07
</td>
<td style="text-align:center;">
9.39
</td>
<td style="text-align:center;">
0.32
</td>
</tr>
</tbody>
</table>

Na tabela acima, os resultados para os anos de 2019 e 2020 estão em
termos de percentuais. De um modo geral, vemos que óbitos por questões
nutricionais acometem, em maior parte, pessoas com baixa escolaridade
(Sem Escolaridade e Ensino Fundamental I), das regiões Nordeste e
Sudeste, do sexo masculino, do grupo de idade mais elevada (mais de 60
anos) e de cor/ra

Para acompanhar a variação anual, foi reportada na coluna Diff a
diferença percentual entre os ans de 2020 e 2019. Em termos relativos,
vemos que houve um aumento para os indivíduos dos 3 níveis educacionais
mais altos (Ensino Médio, Superior Incompleto e Superior Completo) na
variação anual. Ao mesmo tempo, observa-se também um aumento dos Sem
Escolaridade, que é o nível de educação mais baixo registrado. Por outro
lado, indivíduos com Ensino Fundamental I ou II diminuíram a
participação relativa em, respectivamente, 1,31% e 0,08%. Sob o ponto de
vista regional, é possível perceber um importante aumento relativo na
região Nordeste, se tornando a região com maior número de casos de
óbitos por causas nutriciais e, adicionalmente, uma redução de 4% dos
casos na região Sudeste, passando a ser a segunda colocada neste
quesito. Em termos do local de ocorrência, um comportamento perceptível
é o aumento de óbitos dentro de domicílios e redução de registros em
hospitais, o que pode ser certamente explicado pela questâo pandêmica,
uma vez que atendimentos para causas a não ser o da COVID-19 foram
reduzidos significativamente em períodos de pico da pandemia. Sob a
ótica das faixas-etárias, podemos verificar que não ocorreram grandes
variações entre os anos. Por fim, pessoas de cor/raça parda foram mais
acometidas no ano de 2020 enquanto pessoas brancas lideraram este
cômputo no ano anterior.

Vamos verificar graficamente o número de registros por ano

``` r
# Casos de Desnutrição por ano
casos_ano_2020 = Mortalidade_Geral_2020_2019 %>%
  filter(Ano_2020==1) %>%
  filter(Nutricao_1==1)

casos_ano_2019 = Mortalidade_Geral_2020_2019 %>%
  filter(Ano_2019==1) %>%
  filter(Nutricao_1==1)

casos_ano = data.frame(Ano = c(2020, 2019), 
           Casos = c(nrow(casos_ano_2020), nrow(casos_ano_2019)))

ggplot(data = casos_ano, aes(x = as.factor(Ano), y = Casos )) +
  geom_col(stat = 'identity') +
  geom_text(aes(label = Casos), vjust = -0.5) +
  ylim(0,6000) +
  xlab('Ano') +
  theme_bw() +
  ggtitle('Óbitos por Causas Nutricionais (por ano)')
```

![](README_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

De modo geral, vemos que uma queda de 12% nos registros de casos de ob
por causas oriundas da nutrição.

Podemos voltar nossa análise especificamente para o caso da desnutrição
infantil. Deste modo, irei filtar apenas 2 grupos de idade:

-   Menores de 1 ano
-   1 a 4 anos

``` r
# MÉDIAS

# Menos de 1 ano
descritivas_menos = desnutricao %>%
  filter(`FAIXA_ETARIA_Menor que 1 ano`==1) %>%
  summarise_all(mean,na.rm=T)

nomes =  as.data.frame(colnames(descritivas_menos))

descritivas_menos = as.data.frame(as.matrix(t(descritivas_menos)))

descritivas_menos = rownames_to_column(descritivas_menos, 'Variável')

names(descritivas_menos)[2] = 'Média'

descritivas_menos$Variável = gsub(".*_", "", descritivas_menos$Variável)

descritivas_menos$`Média` = round(descritivas_menos$`Média` * 100,2)

descritivas_menos$Grupo = 'Menor que 1 ano'

# 1 a 4 anos
descritivas_1a4 = desnutricao %>%
  filter(`FAIXA_ETARIA_1 a 4 anos`==1) %>%
  summarise_all(mean,na.rm=T)


nomes =  as.data.frame(colnames(descritivas_1a4))

descritivas_1a4 = as.data.frame(as.matrix(t(descritivas_1a4)))

descritivas_1a4 = rownames_to_column(descritivas_1a4, 'Variável')

names(descritivas_1a4)[2] = 'Média'

descritivas_1a4$Variável = gsub(".*_", "", descritivas_1a4$Variável)

descritivas_1a4$`Média` = round(descritivas_1a4$`Média` * 100,2)

descritivas_1a4$Grupo = '1 a 4 Anos'

comparativo = rbind(descritivas_menos, descritivas_1a4)
```

Plotando as informações regionais pelos dois grupos de idade
trabalhados, temos:

``` r
# Gráfico de regiões por grupo de idade
regioes = comparativo[c(8:12,46:50),]

ggplot(regioes, aes(fill=`Variável`, y=`Média`, x=Grupo)) + 
  geom_bar(position="dodge", stat="identity") + 
  theme_bw() +
  xlab('Grupo de Idade') +
  ylab('Percentual') +
  ggtitle('Localização dos Óbitos por Causas Nutricionais (por grupo de idade)')
```

![](README_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

Ao observar o gráfico acima, vemos que existe uma ocorrência
consideravelmente maior de óbitos em crianças menores que 1 ano na
região Nordeste, enquanto que para crianças entre 1 a 4 anos, a região
Norte é mais destacada. Neste sentido, pode-se argumentar que o baixo
desenvolvimento de ambas as regiões parece influenciar diretamente na
vulnerabilidade alimentar dos mais jovens.

Na sequência, apresento os mesmos dados desagregados por raça/cor

``` r
# Gráfico de Cor/Raça por grupo de idade
cor = comparativo[c(30:34,68:72),]

ggplot(cor, aes(fill=`Variável`, y=`Média`, x=Grupo)) + 
  geom_bar(position="dodge", stat="identity") + 
  theme_bw() +
  xlab('Grupo de Idade') +
  ylab('Percentual') +
  ggtitle('Raça/Cor dos Óbitos por Causas Nutricionais (por grupo de idade)')
```

![](README_files/figure-gfm/unnamed-chunk-20-1.png)<!-- --> Desta vez,
também podemos correlacionar questões sociais com os resultados
observado, uma vez que, existe uma proporção muito destacada de pardos e
indígenas nas ocorrências registradas. No caso específico dos casos
registrados por crianças oriundas de populações indígenas, vemos que o
percentual relativo obtido para ambos os grupos (aproximadamente 30%
para o grupo de 1 a 4 anos e 18% menores que 1 ano) estão muito acima da
participação deste grupo étnico na sociedade brasileira, que é de
aproximadamente 2%, segundo dados da PNAD Contínua de 2019.

O gráfico abaixo exibe o registro por local de ocorrência.

``` r
# Gráfico de Local de Ocorrência por grupo de idade
local = comparativo[c(15:21,53:59),]
local = local %>%
  filter(Variável != 'Ignorado')

ggplot(local, aes(fill=`Variável`, y=`Média`, x=Grupo)) + 
  geom_bar(position="dodge", stat="identity") + 
  theme_bw() +
  xlab('Grupo de Idade') +
  ylab('Percentual') +
  ggtitle('Local de Ocorrência dos Óbitos por Causas Nutricionais (por grupo de idade)')
```

![](README_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

É perceptível que para ambos os grupos, há predominância de registros de
óbitos em hospitais para ambos os grupos.

# Análise Inferencial

Tendo em vista o que foi acima apresentado, nesta seção pretende-se
realizar uma análise de fatores relacionados a ocorrência de óbitos
relacionados a questões nutricional. Sendo assim, basicamente, estaremos
estimando o seguinte modelo:

![Y\_{irt} = \\alpha + \\beta X\_{it} + \\lambda_r + \\omega_t \\ +\\varepsilon\_{irt}](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;Y_%7Birt%7D%20%3D%20%5Calpha%20%2B%20%5Cbeta%20X_%7Bit%7D%20%2B%20%5Clambda_r%20%2B%20%5Comega_t%20%5C%20%2B%5Cvarepsilon_%7Birt%7D "Y_{irt} = \alpha + \beta X_{it} + \lambda_r + \omega_t \ +\varepsilon_{irt}")

Especificamente, nossa variável dependente
![Y](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;Y "Y")
é uma dummy que resulta em 1 se o óbito do indivíduo
![i](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;i "i")
na região
![r](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;r "r")
e no ano
![t](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;t "t")
teve óbito registrado tendo causa relacionada a questões nutricionais. O
termo
![\\alpha](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Calpha "\alpha")
é o intercepto da nossa regressão, o vetor
![X_it](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;X_it "X_it")
compreende variáveis importantes para explicar a nossa variável
dependente, tais como sexo e raça,
![\\omega_t](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Comega_t "\omega_t")
é o efeito fixo de tempo que serve para controlar efeitos que ocorreram
ao longo do tempo comum a todas as regiões,
![\\lambda_r](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Clambda_r "\lambda_r")
compreende os efeitos fixos de região que é útil para controlar as
especificidades de cada região no Brasil e, por fim,
![\\varepsilon\_{irt}](https://latex.codecogs.com/png.image?%5Cdpi%7B110%7D&space;%5Cbg_white&space;%5Cvarepsilon_%7Birt%7D "\varepsilon_{irt}")
denota os erros do modelo de regressão.

Primeiramente, vou identificar a relação dos fatores acima citados para
toda a amostra de dados dos anos de 2019 e 2020 (Modelo 1) e,
posteriormente, a amostra será restrita apenas para crianças de até 4
anos (Modelo 2). A estimação será realizada por meio do método de
Mínimos Quadrados Ordinários (MQO), onde, como a variável dependente é
uma dummy, constiui-se então em um Modelo de Probabilidade Linear (MPL)

``` r
modelo_1 = feols(Nutricao_1 ~  REGIAO_Norte  + REGIAO_Nordeste + REGIAO_Sul + `REGIAO_Centro-Oeste` + SEXO_NOVO_Masculino + RACACOR_NOVO_Amarela + RACACOR_NOVO_Indígena + RACACOR_NOVO_Parda + RACACOR_NOVO_Preta + Ano_2020 , se= 'hetero',
                  # cluster = 'mun', 
                  data = Mortalidade_Geral_2020_2019)


modelo_2 = feols(Nutricao_1 ~  REGIAO_Norte  + REGIAO_Nordeste + REGIAO_Sul + `REGIAO_Centro-Oeste` + SEXO_NOVO_Masculino + RACACOR_NOVO_Amarela + RACACOR_NOVO_Indígena + RACACOR_NOVO_Parda + RACACOR_NOVO_Preta + Ano_2020 , se= 'hetero', 
                  subset = Mortalidade_Geral_2020_2019$`FAIXA_ETARIA_Menor que 1 ano`==1 | Mortalidade_Geral_2020_2019$`FAIXA_ETARIA_1 a 4 anos`==1,  
                  # cluster = 'mun', 
                  data = Mortalidade_Geral_2020_2019)
```

``` r
etable(modelo_1, modelo_2, headers = c('Amostra Total', 'Até 4 Anos'))
```

    ##                                   modelo_1            modelo_2
    ##                              Amostra Total          Até 4 Anos
    ## Dependent Var.:                 Nutricao_1          Nutricao_1
    ##                                                               
    ## (Intercept)            0.0036*** (7.84e-5)  0.0030*** (0.0005)
    ## REGIAO_Norte            0.0020*** (0.0002)  0.0061*** (0.0010)
    ## REGIAO_Nordeste         0.0019*** (0.0001)  0.0036*** (0.0007)
    ## REGIAO_Sul             -0.0003** (9.34e-5)    -0.0005 (0.0006)
    ## `REGIAO_Centro-Oeste`   0.0008*** (0.0001)    -0.0006 (0.0009)
    ## SEXO_NOVO_Masculino   -0.0006*** (7.19e-5)    -0.0007 (0.0005)
    ## RACACOR_NOVO_Amarela    0.0023*** (0.0006) -0.0044*** (0.0005)
    ## RACACOR_NOVO_Indígena   0.0144*** (0.0014)  0.0419*** (0.0052)
    ## RACACOR_NOVO_Parda       4.91e-5 (8.57e-5)    -0.0003 (0.0006)
    ## RACACOR_NOVO_Preta      0.0005*** (0.0001)     0.0006 (0.0014)
    ## Ano_2020              -0.0010*** (7.13e-5)    -0.0006 (0.0005)
    ## _____________________ ____________________ ___________________
    ## S.E. type             Heteroskedasti.-rob. Heteroskedast.-rob.
    ## Observations                     2,830,937              70,873
    ## R2                                 0.00056             0.01026
    ## Adj. R2                            0.00056             0.01012
    ## ---
    ## Signif. codes: 0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

Os resultados apresentados na tabela anterior nos informam que, no nosso
modelo 1, onde consideramos a amostra total, indivíduos das regiões
Norte, Nordeste e Centro-Oeste têm, em média, maior probabilidade de
ocorrência de óbito por causas nutricionais quando comparamos com
indivíduos da região Sudeste (dummy de região que foi excluída da
regressão). Cabe resssaltar que a probabilidade é maior para o caso das
regiões Norte e Nordeste (0,2% maior em média). Por outro lado,
indivíduos da região Sul têm menor probabilidade neste sentido, embora
essa diferença seja muito pequena - próxima de 0 - ela é
estatisticamente significativa. Ademais, também podemos destacar que, na
amostra total, observou-se que homens têm menos chances de serem
acometidos por questões nutricionais. Na questão racial, observou-se que
indivíduos indígenas, amarelos e pretos têm maiores probabilidades de
óbito por causa nutricional em comparação com brancos, com destaque aos
primeiros, que têm mais de 1% de chances a mais de terem óbito
registrado por causas nutricionais.

Passando para o modelo 2, onde reduzi a amostra para apenas crianças de
até 4 anos, assim como havia verificado anteriormente, indivíduos das
regiões Norte e Nordeste têm maiores chances de terem óbito por causa
nutricional (0,6% e 0,3% respectivamente ao comparar com a região
sudeste). Entretanto, não foram encontradas diferenças significativas em
relação aos registros das regiões Sul e Centro-Oeste. Aqui também não
foram encontradas diferenças significativas em relação ao Sexo. Assim
como havíamos verificado anteriormente, indígenas tem significativa
maior probabilidade de acometimentos por desnutrição quando comparado
com crianças brancas. Em termos de magnitude, a probabilidade de óbito é
mais de 4% maior do que o de crianças brancas. Foi também verificado que
crianças de raça/cor amarela tem uma pequena probabilidade menor de ter
óbito oriundo de questão nutricional (apenas 0,4% menor), enquanto não
foram encontradas diferenças ao considerar crianças padras e pretas para
este grupo de idade.

# Comentários Gerais

Este exercício buscou fazer uma rápida análise a respeito dos dados do
SIM para os anos de 2019 e 2020, observando especificamente a questão
dos óbitos por causas relacionadas com nutrição no Brasil.

Inicialmente, verificamos uma predominância relativa de acometimentos
por parte de indivíduos menos escolarizados, das regiões Nordeste e
Sudeste, do sexo masculino, com casos ocorrendo em hospitais, de idade
maior de 60 anos e de cor parda e branca. Em termos de variação anual
entre 2020 e 2019, é importante salientar o grande aumento de registros
em domicílio e, ao mesmo tempo, significativa redução de registros em
hospitais, o que pode ser explicado pelas restrições nos atendimentos de
causas a não ser a da pandemia. Em análise gráfica, por outro lado,
mostrada uma redução de 12% nos referidos casos entre 2019 e 2020.

Por fim, foram rodadas algumas regressões com intuito de verificar
relações entre características dem dos indivíduos. Primeiro, foi rodado
um modelo onde foi considerada toda a amostra de dados e, deste modo,
foi observado que, tudo mais constante, há maior probabilidade de
ocorrência de casos nas regiões Norte, Nordeste e Centro-Oeste em
relação a região sudeste e, por outro lado, uma menor probabilidade na
região Sul. Também foi verificado uma menor chance para indivíduos do
sexo masculino e maior probabilidade de acometimento em indivíduos de
cor/raça amarela, indígena e preta em relação a indivíduos brancos.

Quando investigamos nossa subamostra para indivíduos de até 4 anos,
continuamos observando maior probabilidade de ocorrência em crianças das
regiões Norte e Nordeste, com probabilidade maior neste caso. Foi
encontrado também maior probabilidade de indivíduos indígenas terem
óbitos registrados por causas nutricionais em relação a crianças
brancas, por outro lado, crianças de raça/cor amarela tem uma menor
probabilidade neste mesmo sentido.

# Mapa interativo do Programa Bolsa Família (desafio Base dos dados)

Esse projeto tem como inspiração o desafio apresentado pelo site [base dos dados](https://basedosdados.org/) no twitter: https://twitter.com/basedosdados/status/1439989535132299264 .

Utilizando a base de dados referente ao [Programa Bolsa Família](https://www.gov.br/cidadania/pt-br/acoes-e-programas/bolsa-familia) disponível no [site](https://basedosdados.org/dataset/8309d084-35ee-483c-bbad-f41e35ff94c9) e as estimativas populacionais do [IBGE](https://basedosdados.org/dataset/e178b35b-94fe-4021-b82b-cdb90a19a716) criarei um mapa interativo demonstrando a evolução do número de famílias atendidas pelo programa em relação à população por estado.

 

## Índice

- [Pacotes](#pacotes)
- [Dados](#dados)
- [Análise](#análise)
- [Mapa](#mapa)



## Pacotes


-   [Pandas](http://pandas.pydata.org/)
-   [basedosdados](https://basedosdados.github.io/mais/access_data_packages/)
-   [matplotlib](https://matplotlib.org/)
-   [json](https://docs.python.org/3/library/json.html)
-   [plotly](https://plotly.com/)



## Dados

Os dados foram obtidos por meio do site [base dos dados](https://basedosdados.org/) e os dados de georreferenciados do [GIS Dataset Brasil](https://fititnt.github.io/gis-dataset-brasil/)

-   **br_mc_indicadores**: dados referentes ao bolsa família (obtido diretamente atravez do data lake) 

![image](https://user-images.githubusercontent.com/48839817/135153131-3e88c8ba-02c8-4188-bf8f-ec92db8fa823.png)


-   **municipio.csv**: Dados populacionais 

![image](https://user-images.githubusercontent.com/48839817/135153252-66193869-a63b-41d7-9fe3-237e7276b4ff.png)


## Análise

A primeira necessidade é criar uma coluna para referenciar o código da UF (utilizando os dois primeiros dígitos do ID municipal):

```
df2["COD_UF"] = df2["id_municipio"].str[0:2]
```

Além disso, precisamos transformar as colunas familias_beneficiarias_pbf, id_municipio (esta coluna também será modificada no DF de municípios) e COD_UF para o tipo int. Após isso selecionarei apenas o mês 1 (Janeiro) como referência - filtrando o DF.

Para adicionar os dados populacionais provenientes do DF municipal no DF principal Utilizei uma lista para armazenar os dataframes filtrados por ano. Essa lista será alimentada com o dataframe principal filtrado por ano e com uma coluna referente à população adquirida a partir de um dicionário com as chaves como id e o valor como a populaçõa do segundo dataframe:

```
lista = []
for i in range(2004,2021):
    #filtrando os dataframes
    filtered_mun = mun[mun['ano']== i]
    filtered_mensal = mensal[mensal['ano']== i]
    
    #Criando o dicionário
    pop_dict = dict(zip(filtered_mun["id_municipio"], filtered_mun['populacao']))
    
    #Criando a nova coluna e appendando à lista
    filtered_mensal["POPULAÇÃO ESTIMADA"] = filtered_mensal["id_municipio"].map(pop_dict)
    lista.append(filtered_mensal)
    
 ```
Após a concatenação o dataframe resultante:

![image](https://user-images.githubusercontent.com/48839817/135158011-7b23da2c-7ad6-4246-b277-b3f56c71cf40.png)

Finalmente o processo de transformação dos dados é concluido com a adição da coluna com a sigla dos estados e a seleção dos dados no formato a ser usado no gráfico:

```
# Lista com as  UFs e seus códigos
estados  =  {11: "RO", 12: 'AC',13:"AM",14:"RR",15:"PA",16:"AP",17:"TO",21:"MA",22:"PI",23:"CE",
             24:"RN",25:"PB",26:"PE",27:"AL",28:"SE",29:"BA",31:"MG",32:"ES",33:"RJ",35:"SP",
             41:"PR",42:"SC",43:"RS",50:"MS",51:"MT",52:"GO",53:"DF"}
             
#Criação da coluna UF com base no dicionário
concated['COD_UF'] = concated['COD_UF'].astype(int)
concated['UF'] = concated['COD_UF'].map(estados)


# Selecionando apenas as colunas necessárias para a análise
to_group = concated[["ano", 'UF' ,'familias_beneficiarias_pbf','POPULAÇÃO ESTIMADA' ]]

# Agrupando os dados por ano e unidade federativa
grouped = to_group.groupby(by=['UF', 'ano']).sum().reset_index()

```

![image](https://user-images.githubusercontent.com/48839817/135158554-a401a06b-3711-4ce6-a4ea-74d49b860814.png)

Por fim criamos a coluna relativa:

```
# Criando uma coluna com os valores relativos 
grouped['pbf_relativo'] = grouped["familias_beneficiarias_pbf"]/ grouped['POPULAÇÃO ESTIMADA']
```

O arquivo JSON precisa de uma correção para funcionar com o mapa:

```
# Para a leitura do mapa é necessário que a chave id tenha o mesmo valor da base.
# Para isso iterei pela chave features e inseri em id o valor da cheve 'UF_05'
for i in range(len(brasil['features'])):
    brasil['features'][i]['id'] = brasil['features'][i]['properties']['UF_05']
```

## Mapa

Com os dados preparados o mapa é facilmente criado com esse código (referência  https://python.plainenglish.io/how-to-create-a-interative-map-using-plotly-express-geojson-to-brazil-in-python-fb5527ae38fc):

```

fig = px.choropleth_mapbox(
grouped, # selecionando a base
mapbox_style = "carto-positron", #selecionando o tipo de cartografia
locations = 'UF', # A coluna do DF que será relacionada com JSON
geojson = brasil, #A fonte do JSON
color = "pbf_relativo", #Valores que definirão a cor do gráfico
hover_name = 'UF', #Informação que aparecerá no box
hover_data = ["familias_beneficiarias_pbf"], # Informação extra para aparecer no box
title = "Percentual de famílias beneficiárias do PBF em relação à população (estimativa IBGE 2021) por estado", # Titulo
center={"lat": -14, "lon": -55}, # Centralização inicial do gráfico
zoom=2,  # Zoom inicial do gráfico
animation_frame = 'ano', # Coluna que será iterada para a animação
opacity = 0.6 )

fig.update_layout(height=800, width = 1000)
fig.show()
```

![pbf](https://user-images.githubusercontent.com/48839817/135160409-54314917-4708-4798-90f2-e00a067d3ae2.gif)



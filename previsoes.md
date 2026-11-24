## Parte A — MongoDB

### A1. Painel do pátio

Previsão da A: Como não tem índice, o banco provavelmente vai rodar um COLLSCAN, lendo ele todo para achar a informação desejada

Previsão da B: Com a implementação de indices, vai ser um IXSCAN, vai ler bem menos documentos e ainda deve ordenar por causa do filtro de data

Previsão da C: Outro IXSCAN, só que dessa vez não vai ordernar por data 
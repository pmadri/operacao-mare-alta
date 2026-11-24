## A1

a) 

criadodb.movimentacoes.find({
  terminal: "T5",
  classe: "Refrigerado",
  ts: { $gte: ISODate("2026-04-01T00:00:00.000Z") }
}).sort({ peso_kg: -1 }).limit(20).explain("executionStats")

> Estatísticas
stage: 'COLLSCAN'
totalDocsExamined: 400000
totalKeysExamined: 0
nReturned: 20
executionTimeMillis: 659

---------------------------------------------------------------

b) 

db.movimentacoes.createIndex(
  { terminal: 1, classe: 1, ts: 1, peso_kg: -1 },
  { name: "idx_a1_ordem_filtro" }
)

db.movimentacoes.find({
  terminal: "T5",
  classe: "Refrigerado",
  ts: { $gte: ISODate("2026-04-01T00:00:00.000Z") }
}).sort({ peso_kg: -1 }).limit(20).explain("executionStats")

> Estatísticas
nReturned: 20
executionTimeMillis: 18
totalDocsExamined: 20
totalKeysExamined: 13754
stage: 'IXSCAN'
stage: 'FETCH'

---------------------------------------------------------------

c) 

db.movimentacoes.dropIndex("idx_a1_ordem_filtro")

db.movimentacoes.createIndex(
  { terminal: 1, classe: 1, peso_kg: -1, ts: 1 },
  { name: "idx_a1_esr" }
)

db.movimentacoes.find({
  terminal: "T5",
  classe: "Refrigerado",
  ts: { $gte: ISODate("2026-04-01T00:00:00.000Z") }
}).sort({ peso_kg: -1 }).limit(20).explain("executionStats")

> Estatísticas
executionTimeMillis: 5
nReturned: 20
totalKeysExamined: 27
totalDocsExamined: 20
stage: 'IXSCAN'
stage: 'FETCH'
stage: 'LIMIT'

-------------------------------------------------------------- 

d)  EXPLICAÇÃO E DIFERENÇAS

a) Sem índice: lê a coleção inteira (COLLSCAN) e ordena na memória (SORT), examinando perto de 400.000 documentos e 0 chaves.

b) { terminal: 1, classe: 1, ts: 1, peso_kg: -1 }: a B-Tree ordena por data (ts) antes do peso (peso_kg). Como há várias datas no intervalo ($gte), o peso não fica em ordem geral entre elas, então o MongoDB precisa fazer SORT na memória.

c) { terminal: 1, classe: 1, peso_kg: -1, ts: 1 } (ESR): igualdade (terminal, classe) vem primeiro, depois ordenação (peso_kg: -1) e por último intervalo (ts: 1). O índice já vem na ordem decrescente de peso, dispensando o SORT e parando assim que achar 20 itens válidos.

-------------------------------------------------------------

## A2

1. Comportamento antes do índice (erro esperado)

db.movimentacoes.find({
  posicao: {
    $nearSphere: {
      $geometry: {
        type: "Point",
        coordinates: [-48.49, -25.458]
      },
      $maxDistance: 300
    }
  }
}).limit(10)

> Erro esperado:

Uncaught:
MongoServerError[NoQueryExecutionPlans]: error enumerating plans for query: ns=logiporto.movimentacoes limit=10Tree: GEONEAR  field=posicao maxdist=300 isNearSphere=0
Sort: {}
Proj: {}
 planner returned error :: caused by :: unable to find index for $geoNear query

-----------------------------------------------------------------

2. Criação do índice 2dsphere e execução com .explain("executionStats")

db.movimentacoes.createIndex(
  { posicao: "2dsphere" },
  { name: "idx_posicao_2dsphere" }
)

db.movimentacoes.find({
  posicao: {
    $nearSphere: {
      $geometry: {
        type: "Point",
        coordinates: [-48.49, -25.458]
      },
      $maxDistance: 300
    }
  }
}).limit(10).explain("executionStats")

------------------------------------------------------------

3. Explicação de cada etapa do plano

Estágio geoespacial (2dsphere / GEO_NEAR_2DSPHERE): olha o mapa/índice em posicao para achar quem está a até 300 metros do ponto do incidente e já coloca do mais perto para o mais longe.

FETCH: pega o documento completo de cada posição encontrada para mostrar os dados da movimentação.

LIMIT: para assim que entregar no máximo 10 resultados.

------------------------------------------------------------

## A3 

1. Comprovação no banco

db.alertas.findOne({}, { _id: 0, registrado_em: 1, tipoRegistradoEm: { $type: "$registrado_em" } })
db.alertas.getIndexes()


Resultado: $type retorna "string", comprovando que registrado_em foi salvo como texto (.toISOString()) e não como BSON Date.

2. Causa raiz

A thread de expiração TTL do MongoDB só calcula o vencimento em campos indexados cujo valor seja do tipo BSON Date (ou array de Date). Quando o valor é String, o MongoDB não converte texto para data e simplesmente ignora o documento sem gerar erro, fazendo com que nenhum alerta antigo seja removido.

3. Comando do índice TTL correto (30 dias = 2592000 segundos)


-------------------------------------------------------

## A4
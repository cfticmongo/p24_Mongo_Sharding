# Introducción a Sharding

Arquitctura distribuida de los cluster de servidores en la que los datos de algunas colecciones se
reparten entre los diferentes shards (particiones) para escalar horizontalmente nuestros sistemas.

- ESCALADO HORIZONTAL

## Componentes del sharding

- Shards. Cada una de las particiones en las que se distribuyen los datos (cluster replica set)
- Config server. Cluster con los metadatos de los shards (cluster replica set)
- Mongos (router). Servidor o servidores, que enrutan las operaciones a cada shard.

- Chunk. Mecanismo automatizado de migración de datos de un shard a otro para balancear el almacenamiento
de cada cluster.
# Primario en los shard cluster

¡¡¡El primario en un shard cluster no es los mismo que el primario de un replica set!!!!

El shard primary de un shard cluster (NO CONFUNDIR CON EL PRIMARY DE UN REPL SET) es el que
aloja todas las colecciones no particionadas.

El shard primary se establece a nivel de base de datos lo que implica que cada base de datos tendrá
un shard (y no tiene que ser el mismo que para otra base de datos) en el que almacene todas las
colecciones de esa base de datos que no se hayan particionado.

¿Como selecciona o determina MongoDB el shard primary de cada base de datos?

Cuando se crea la base de datos selecciona el shard ¿que contenga menos datos?

¿Como sabemos de una base de datos cual es es su shard primary?

En la base de datos config del configServer

Desde una shell conectada a mongos

use config

db.databases.find() // Devuelve info de la base de datos e indicada de cada una su  shard primary

Si añadimos una colección no particionada

use maraton

db.jueces.insert({nombre: "Sara"})

Desde mongos veremos todas las colecciones

Pero si probamos desde los primarios de cada shard, la coleccion jueces solo se verá desde
el shard primary de esa base datos maraton.


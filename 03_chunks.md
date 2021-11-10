# Chunk en sharding

Un chunk es una agrupación lógica de los documentos de una colección sharding, con un rango para distribuir uniformemente
los documentos en los diferentes shard con el objetivo de minimizar el impacto de las distribuciones.

Almacena documentos hasta un determinado tamaño indicado en el chunkSize (por defecto es 64MB). Cuando el tamaño del chunk
supera ese valor se divide automaticamente (split), generando un nuevo chunk y diviendo los rangos correspondientes.

Cuando se produce una diferencia entre el nº de chunks de un shard y otro, si el balanceador está activado, se produce una
migración automática del chunk del shard más próximo en rango a los rangos del shard de destino.

Esta descompensación lanza la migración, si el balanceador está activado, dependiendo del número toral de
chunks generados en todos los shard:

nº Chunk totales        Diferencia chuncks/sharding que genera migración

menor 20                        2
20-79                           4
más 80                          8

# Modificación del tamaño del chunk por defecto

Si queremos una mejor distribución de la colección podemos reducir el tamaño, el inconveniente es que, si tenemos activado
constantemente el balanceador, habrá más migraciones.

1.- Desde mongos

use config

db.settings.save({_id: "chunkSize", value: 4}) // Tamaño en megas

# Procedimiento de migración

1.- Inicio del proceso, las operaciones sobre documentos del chunk se mantienen en el shard de origen.

2.- Se crea el nuevo chunk en el shard de destino y se realiza una actualización del índice con los nuevos
documentos en el shard de destino.

3.- La escritura de los documentos en el shard de destino.

4.- Sincronización del chunk de destino con el origen por si se producen actualizaciones durante el proceso.

5.- Actualiza la info en el configserver de los chunk afectados.

6.- Las operaciones de los documentos de ese chunk pasan al nuevo shard destino y el chunk del shard de origen es eliminado.

# Balancer

El balancer es un mecanismo de los cluster sharding ue realiza las mmigraciones de manera automática de acuerdo
a los criterios anteriores. 

Disponemos de los siguientes métodos para administrar el balancer (por defecto está activado):

sh.startBalancer() // Activa el balanceador

sh.stopBalancer() // Desactiva el balanceador

En ventanas de mantenimiento

db.settings.update( 
   { _id: "balancer" },
   { $set: { activeWindow : { start : "<start-time>", stop : "<stop-time>" } } }, // time en HH:MM
   { upsert: true }
)

sh.getBalancerState() // Devuelve el estado si está activado o no

sh.isBalancerRunning() // Devuelve si se están produciendo migraciones en ese momento

## Jumbo chunks

Estado de un chunk (en los reportes de sharding nos indicará si hubiera alguno) cuando supera
el tamaño máximo chunkSize pero no puede dividirse automáticamente porque el rango que
usa la coleccion para sharding queda limitado a un valor.

En el caso de los jumbo chunks Mongo permite mantenerlos pero los marca como jumbo y no se podrán migrar a otros
shard con lo cual puede provocar una descompensación de la colección.


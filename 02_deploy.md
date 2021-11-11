# Deploy de sharding cluster

## Deploy del config server

Necesitamos un replica set para este config server de 3 miembros

1.- Crear directorios para cada miembro

mkdir configServer1
mkdir configServer2
mkdir configServer3

2.- Levantar los 3 servidores del cluster

mongod --replSet configServerGetafe --dbpath configServer1 --port 27001 --configsvr
mongod --replSet configServerGetafe --dbpath configServer2 --port 27002 --configsvr
mongod --replSet configServerGetafe --dbpath configServer3 --port 27003 --configsvr

3.- Lanzar la configuración del cluster

Nos conectamos con la shell a uno de ellos

mongo --port 27001

Inicializar el replica set:

rs.initiate({
    _id: "configServerGetafe",
    members: [
        {_id: 0, host: "localhost:27001"},
        {_id: 1, host: "localhost:27002"},
        {_id: 2, host: "localhost:27003"},
    ],
    configsvr: true // Esta propiedad determina que el cluster se use como config server de un shard cluster
})


## Deploy de los shards (Creamos 2 shard)

1.- Directorios del primer shard A (replica set)

mkdir shardAServer1
mkdir shardAServer2
mkdir shardAServer3

2.- Levantamos los servidores del primer shard A

mongod --replSet shardAServerGetafe --dbpath shardAServer1 --port 27101 --shardsvr
mongod --replSet shardAServerGetafe --dbpath shardAServer2 --port 27102 --shardsvr
mongod --replSet shardAServerGetafe --dbpath shardAServer3 --port 27103 --shardsvr

3.- Inicializamos el replica set

mongo --port 27101

rs.initiate({
    _id: "shardAServerGetafe",
    members: [
        {_id: 0, host: "localhost:27101"},
        {_id: 1, host: "localhost:27102"},
        {_id: 2, host: "localhost:27103"},
    ]
})

4.- Directorios del segundo shard B (replica set)

mkdir shardBServer1
mkdir shardBServer2
mkdir shardBServer3

5.- Levantar los servidores del segundo shard

mongod --replSet shardBServerGetafe --dbpath shardBServer1 --port 27201 --shardsvr
mongod --replSet shardBServerGetafe --dbpath shardBServer2 --port 27202 --shardsvr
mongod --replSet shardBServerGetafe --dbpath shardBServer3 --port 27203 --shardsvr

6.- Inicializar el replica set del segundo shard 

mongo --port 27201

rs.initiate({
    _id: "shardBServerGetafe",
    members: [
        {_id: 0, host: "localhost:27201"},
        {_id: 1, host: "localhost:27202"},
        {_id: 2, host: "localhost:27203"},
    ]
})

## Mongos (router)

1.- Levantar mongos con comando

mongos --configdb configServerGetafe/localhost:27001,localhost:27002,localhost:27003 --port 27000

2.- Añadir los shard al cluster

Nos conectamos con una shell a mongos

mongo --port 27000

Para añadir cada shard

sh.addShard("shardAServerGetafe/localhost:27101,localhost:27102,localhost:27103")
sh.addShard("shardBServerGetafe/localhost:27201,localhost:27202,localhost:27203")

Chequeamos con

sh.status()

  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("618bec1cda51c5bccadc70e3")
  }
  shards:
        {  "_id" : "shardAServerGetafe",  "host" : "shardAServerGetafe/localhost:27101,localhost:27102,localhost:27103",  "state" : 1 }
        {  "_id" : "shardBServerGetafe",  "host" : "shardBServerGetafe/localhost:27201,localhost:27202,localhost:27203",  "state" : 1 }
  active mongoses:
        "4.2.3" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:


## Crear base de datos con colección particionada

No todas las bases de datos y las colecciones de un shard estarán particionadas.

1.- Crear una base de datos que permita el particionado de sus colecciones

Desde la shell conectada a mongos usamos el método sh.enableSharding(<nombre-base-datos>)

```
sh.enableSharding("maraton")
```

2.- Creamos, en esa base de datos, una colección particionada usando el método sh.shardCollection(<namespace.col>, {<shard-keys>})

Por ejemplo:

```
sh.shardCollection("maraton.runners", {surname1: 1, age: 1}) // Las shard keys usan la misma sintaxis de los índices
```

3.- Para comprobar la repartición de los datos en cada shard vamos a insertar un grupo grande de datos

let names = ['Laura','Juan','Fernando','María','Carlos','Lucía','David'];
let surnames = ['Fernández','Etxevarría','Nadal','Novo','Sánchez','López','García'];
let dniWords = ['A','B','C','D','P','X'];
let runners = [];
for (i = 0; i < 1000000; i++) {
    runners.push({
        name: names[Math.floor(Math.random() * names.length)],
        surname1: surnames[Math.floor(Math.random() * surnames.length)],
        surname2: surnames[Math.floor(Math.random() * surnames.length)],
        age: Math.floor(Math.random() * 100),
        dni: Math.floor(Math.random() * 1e8) + dniWords[Math.floor(Math.random() * dniWords.length)]
    })
}

use maraton

db.runners.insert(runners)

4.- Comprobamos la distrubución en los shard de los documentos

db.runners.getShardDistribution()

Shard shardAServerGetafe at shardAServerGetafe/localhost:27101,localhost:27102,localhost:27103
 data : 38.75MiB docs : 345062 chunks : 2
 estimated data per chunk : 19.37MiB
 estimated docs per chunk : 172531

Shard shardBServerGetafe at shardBServerGetafe/localhost:27201,localhost:27202,localhost:27203
 data : 71.07MiB docs : 654938 chunks : 3
 estimated data per chunk : 23.69MiB
 estimated docs per chunk : 218312

Totals
 data : 109.83MiB docs : 1000000 chunks : 5
 Shard shardAServerGetafe contains 35.28% data, 34.5% docs in cluster, avg obj size on shard : 117B
 Shard shardBServerGetafe contains 64.71% data, 65.49% docs in cluster, avg obj size on shard : 113B

sh.status()

--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("618bec1cda51c5bccadc70e3")
  }
  shards:
        {  "_id" : "shardAServerGetafe",  "host" : "shardAServerGetafe/localhost:27101,localhost:27102,localhost:27103",  "state" : 1 }
        {  "_id" : "shardBServerGetafe",  "host" : "shardBServerGetafe/localhost:27201,localhost:27202,localhost:27203",  "state" : 1 }
  active mongoses:
        "4.2.3" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                2 : Success
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shardAServerGetafe	1
                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : shardAServerGetafe Timestamp(1, 0) 
        {  "_id" : "maraton",  "primary" : "shardBServerGetafe",  "partitioned" : true,  "version" : {  "uuid" : UUID("4a989847-68be-496f-838b-6cb5264ffa9e"),  "lastMod" : 1 } }
                maraton.runners
                        shard key: { "surname1" : 1, "age" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                shardAServerGetafe	2
                                shardBServerGetafe	3
{ "surname1" : { "$minKey" : 1 }, "age" : { "$minKey" : 1 } } -->> { "surname1" : "Etxevarría", "age" : 0 } on : shardAServerGetafe Timestamp(3, 0) 
{ "surname1" : "Etxevarría", "age" : 0 } -->> { "surname1" : "García", "age" : 42 } on : shardAServerGetafe Timestamp(5, 0) 


{ "surname1" : "García", "age" : 42 } -->> { "surname1" : "Nadal", "age" : 80 } on : shardBServerGetafe Timestamp(4, 2) 
{ "surname1" : "Nadal", "age" : 80 } -->> { "surname1" : "Novo", "age" : 87 } on : shardBServerGetafe Timestamp(4, 3) 
{ "surname1" : "Novo", "age" : 87 } -->> { "surname1" : { "$maxKey" : 1 }, "age" : { "$maxKey" : 1 } } on : shardBServerGetafe Timestamp(5, 1) 

## Añadir un nuevo shard al cluster

Como el objetivo del sharding es el escalado horizontal en todo momento se pueden añadir shards al cluster.

1.- Crear los directorios del 3er shard

mkdir shardCServer1
mkdir shardCServer2
mkdir shardCServer3

2.- Levantar los servidores del tercer shard

mongod --replSet shardCServerGetafe --dbpath shardCServer1 --port 27301 --shardsvr
mongod --replSet shardCServerGetafe --dbpath shardCServer2 --port 27302 --shardsvr
mongod --replSet shardCServerGetafe --dbpath shardCServer3 --port 27303 --shardsvr

3.- Inicializar el replica set del tercer shard

mongo --port 27301

rs.initiate({
    _id: "shardCServerGetafe",
    members: [
        {_id: 0, host: "localhost:27301"},
        {_id: 1, host: "localhost:27302"},
        {_id: 2, host: "localhost:27303"},
    ]
})

4.- Añadir el tercer shard

Desde una shell conectada a mongos:

mongo --port 27000

sh.addShard("shardCServerGetafe/localhost:27301,localhost:27302,localhost:27303");

Comprobar como se produce la migración

sh.isBalacerRunning()  // Comprobar como migrará los chunks

sh.status() // Ver la nueva distribución de los chunks

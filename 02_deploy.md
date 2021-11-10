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
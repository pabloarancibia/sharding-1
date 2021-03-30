# sharding-1
TP1 - parte 2 - Sharding 

//Levanto las instancias con --shardsvr
mongod --shardsvr --dbpath /media/documentoslinux/iot/gvd/tp1/sh/0 --port 27060 --smallfiles --nojournal

//Creamos la carpeta media/documentoslinux/iot/gvd/tp1 (En donde se guarda la Info del Config Server)

//Levanto un Config Server
mongod --dbpath /media/documentoslinux/iot/gvd/tp1/sh/c --configsvr --replSet c --port 27500
mongo --port 27500
	rs.initiate({
		_id: "c",
		members:[{
			_id: 0,
			host: "localhost:27500"
		}]
	})

//Levanto el Servidor de Ruteo (mongos)
// (no funciono asi: mongos --configdb c/localhost:27500 )
// Agrego el puerto
mongos --configdb c/localhost:27500 --port 47017

//Nos conectamos al mongos
mongo --host localhost --port 47017


//Realizamos luego el agregado inicial del shard
	sh.addShard("localhost:27060")

	use finanzas
//Creo el indice
	db.facturas.createIndex({"cliente.region": 1, condPago: 1, _id: 1})

//Activo el Sharding para la base de datos finanzas
	sh.enableSharding("finanzas")

//Shardeo la Colección facturas de la Base de datos Finanzas
	sh.shardCollection("finanzas.facturas", {"cliente.region": 1, condPago: 1, _id: 1})

//Veo la metadata del cluster
	sh.status()

//Veo los chunks que se crearon
	use config
	db.chunks.find({}, {min:1,max:1,shard:1,_id:0,ns:1}).pretty()

//Agrego 2 nuevos shards
mongod --shardsvr --dbpath /media/documentoslinux/iot/gvd/tp1/sh/1 --port 27061 --nojournal
mongod --shardsvr --dbpath /media/documentoslinux/iot/gvd/tp1/sh/2 --port 27062 --nojournal


//
mongo
	sh.addShard("localhost:27061")
	sh.addShard("localhost:27062")

	for (var i=0; i<5; i++) {
		load("/media/documentoslinux/iot/gvd/tp1/facts.js")
	}

	sh.status()

//Query que apunte a un grupo limitado de Shards
	db.facturas.find({"cliente.region":"CABA", "condPago":"30 Ds FF"}).explain()

//Query que se dispara a todos los Shards
	db.facturas.find({"cliente.apellido":"Manoni"}).explain()

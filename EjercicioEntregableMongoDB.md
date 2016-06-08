

#**Actividad1**

Para iniciar la base de datos he ejecutado use ventaParticulares y he rellenado la base de datos con los items.
Para introducir un item:

	db.items.insert({
	descripcion: "Iphone5",
	fecha: new Date ("2016, 5, 2"),
	precio: 50,
	tags: ["telefono movil"],
	vendedor: {email: "ffernandez@gmail.com", psw: "ffernandez"},
	localizacion: {longitude: 38.743671, latitude: -10.552276},
	estado: "disponible"
	})

Aparece confirmación: 

	WriteResult({ "nInserted" : 1 })

Puedo ver todos los items con:
	
	db.items.find().pretty()


# **Primer ejercicio**

**Actualizar la colección “ítems” para hacer una contra-oferta al primer producto disponible que esté etiquetado como “teléfono móvil” y que haya sido puesto en venta con posterioridad al 1/1/2014.**

Declaro una variable de fecha inicial. A continuación añado un subdocumento al primer documento que tenga etiqueta "telefono movil" y fecha mayor que la fecha inicial.
En el subdocumento informo email, pasword, la oferta de 45 euros y su fecha.

	var start = new Date(2014, 1, 1)

	db.items.update({tags:"telefono movil","fecha": {$gt: start}},  
	{$addToSet: {  contraoferta: {email: "ggomez@gmail.com", psw: "ggomez",  oferta: 45, fecha: new Date("2016, 5, 9")}}})

Aparece confirmación:

	WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 0 })


# **Segundo ejercicio**

**Actualizar la colección “ítems” para modificar el estado de todos los productos puestos en venta antes de 1/1/2012 y cuyo estado sea disponible. El nuevo estado pasará a ser descatalogado.**

Declaro una veriable de fecha límite. A continuación actualizo estado a "descatalogado" de todos los documentos (por eso uso $multi=true) cuyo estado es "disponible" y fecha es menor que la fecha límite.  

	var start = new Date(2012, 1, 1)

	db.items.update({estado:"disponible","fecha": {$lt: start}},
	{$set:{estado:"descatalogado"}},
	{$multi:true})

No se han encontrado productos tan antiguos:

	WriteResult({ "nMatched" : 0, "nUpserted" : 0, "nModified" : 0 })


#**Tercer ejercicio**

**Recuperar la descripción y precio de todos los productos etiquetados como “entretenimiento”, cuyo estado sea disponible y que estén en venta en un punto cercano al nuestro (+- 1000 mts.)**

Busco con comando find los items etiquetados como "entretenimiento", con estado "disponible" y cuya localización esté cerca (max mil metros) de las coordenadas específicas.
Selecciono descripción y precio y elimino el id de la selección. Al final pongo .pretty() para que la salida tenga formato.

	db.items.find(
	{tags: "entretenimiento", estado:"disponible","localizacion":
		{$near:
		{$geometry:{type:"Point",coordinates:[ 37.743670, -2.552276]},$maxDistance:1000}
		}
	},
	{_id:0,descripcion:1,precio:1}
	).pretty()
	
He encontrado un producto:

	{ "descripcion" : "Mando xBox negro", "precio" : 10 }


#**Cuarto ejercicio**

**Averiguar el número de ítems disponibles que estén etiquetados como “teléfono móvil” y que cuesten menos de 60€.**

Busco los items etiquetados como "telefono movil", estado "disponible" y precio menos que 60. Solo selecciono ID porque quiero unicamente contar con la función count.

	db.items.find(
	{tags: "telefono movil", precio:{$lt:60}, estado: "disponible"},
	{_id:1}).count()

También puedo escribir la secuencia de otra forma:

	db.items.count( 
	{tags: "telefono movil", precio:{$lt:60}, estado: "disponible"}, 
	{_id:1}
	)

Encuentro dos:

	2

(esto es porque había introducido dos telefonos: Iphone5 e Ipnone6).

#**Quinto ejercicio**

**Eliminar los registros cuyo estado sea “vendido” y que hubieran sido puestos a la venta antes del 1/1/2012.**

Declaro una variable para la fecha limite y a continuación 

	var start = new Date(2012, 1, 1)

	db.items.remove(
	{estado:"vendido","fecha": {$lt: start}},
	{$multi:true})

No se han encontrado ningunos items tan viejos:

	WriteResult({ "nRemoved" : 0 })

#**Actividad2**

Dado que la base de datos debe servir para consultar información relacionada con las tesis y con los investigadores voy a diseñar dos tipos de colecciones: Tesis e Investigadores.

En el agregado **Tesis** se econtrará información basica sobre la misma que refleje el contenido y sirva para facilitar la búsqueda:
el título de la tesis, su autor, palabras clave, codificación UNESCO, instirución y fecha.

En el agregado **Investigadores** estará la información sobre los autores y directores.
Asumo que la misma persona podría ser autor o director en distintos periodos de tiempo.
Por ello diseño una colección que permita identificar el rol (autor/director) y, en consecuencia, contenga información adicional sobre la institución y fecha en la que se desempaña el rol y la tesis (presentada o dirigida).
Si el investigador de la colección es un autor, se podrá encontrar información adicional sobre el director de la tesis.
Si el investigador es director, en este caso la información adicional dará detalles sobre el autor de la tesis dirigida.

El siguiente esquema refleja el **diseño en forma gráfica:**
 
![Doctorados] (https://github.com/annalawrenc/Mongo/blob/master/DisenoMongoTesis2.jpg)

**Implementación**

Creo una base de datos "Doctorados"

	use Doctorados

	switched to db Doctorados

Inserto un ejemplo del documento tipo "Tesis":

	db.tesis.insert({
	autor:{nombre:"Mario", 
	apellidos: "Lopez"},
	titulo:"Elaboración de helados de fresa",
	clave:["helados","fresas"],
	clasificacion_unesco: {clave_un_6dig:"123456", etiqueta_un:"alimentación"},
	fecha_tesis: new Date ("2013, 10, 2"),
	institucion_tesis:"Universidad de Alcalá"
	})

	WriteResult({ "nInserted" : 1 })

Inserto un ejemplo del documento tipo "Investigador". 
En este caso el investigaror es autor de a tesis y aporto información adicional sobre el director.

	db.investigadores.insert({
	nombre:"Mario", 
	apellidos: "Lopez",
	rol: {autor_director:"autor", institucion_investigador: "Universidad de Alcalá", fecha_rol: new Date ("2013, 10, 2") },
	tesis: {titulo:"Elaboración de helados de fresa", etiqueta_un:"alimentación", 
	director:{nombre: "Hugo",apellidos: "Martinez"},institucion_tesis: "Universidad de Alcalá"}
	})

	WriteResult({ "nInserted" : 1 })

Inserto otro ejemplo del documento tipo "Investigador". 
En este caso el investigaror es director y aporto información adicional sobre el autor de la tesis.

	db.investigadores.insert({
	nombre:"Hugo", 
	apellidos: "Martinez",
	rol: {autor_director:"director", institucion_investigador: "Universidad Complutense", fecha_rol: new Date ("2013, 10, 2") },
	tesis: {titulo:"Elaboración de helados de fresa", etiqueta_un:"alimentación", 
	autor:{nombre: "Mario",apellidos: "Lopez"},institucion_tesis: "Universidad de Alcalá"}
	})

	WriteResult({ "nInserted" : 1 })

Compruebo si las inserciones han sido correctas:

	db.tesis.find().pretty()

	{
		"_id" : ObjectId("575756eb2d9d3a4ca7c173c2"),
		"autor" : {
			"nombre" : "Mario",
			"apellidos" : "Lopez"
		},
		"titulo" : "Elaboración de helados de fresa",
		"clave" : [
			"helados",
			"fresas"
		],
		"clasificacion_unesco" : {
			"clave_un_6dig" : "123456",
			"etiqueta_un" : "alimentación"
		},
		"fecha_tesis" : ISODate("2013-10-01T22:00:00Z"),
		"institucion_tesis" : "Universidad de Alcalá"
	}


	db.investigadores.find().pretty()
	{
		"_id" : ObjectId("575753662d9d3a4ca7c173c0"),
		"nombre" : "Mario",
		"apellidos" : "Lopez",
		"rol" : {
			"autor_director" : "autor",
			"institucion_investigador" : "Universidad de Alcalá",
			"fecha_rol" : ISODate("2013-10-01T22:00:00Z")
		},
		"tesis" : {
			"titulo" : "Elaboración de helados de fresa",
			"etiqueta_un" : "alimentación",
			"director" : {
				"nombre" : "Hugo",
				"apellidos" : "Martinez"
			},
			"institucion_tesis" : "Universidad de Alcalá"
		}
	}


	{
		"_id" : ObjectId("5757537f2d9d3a4ca7c173c1"),
		"nombre" : "Hugo",
		"apellidos" : "Martinez",
		"rol" : {
			"autor_director" : "director",
			"institucion_investigador" : "Universidad Complutense",
			"fecha_rol" : ISODate("2013-10-01T22:00:00Z")
		},
		"tesis" : {
			"titulo" : "Elaboración de helados de fresa",
			"etiqueta_un" : "alimentación",
			"autor" : {
				"nombre" : "Mario",
				"apellidos" : "Lopez"
			},
			"institucion_tesis" : "Universidad de Alcalá"
		}
	}


Adicionalemnte voy a crear índices según los criterios de búsqueda más comunes:

Para las busquedas por area de conocimiento indexo la etiqueta unesco.


	db.tesis.ensureIndex({"clasificacion_unesco.etiqueta_un":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 1,
		"numIndexesAfter" : 2,
		"ok" : 1
	}

Para las busquedas por institución de la tesis indexo nombre de instirución.


	db.tesis.ensureIndex({"institucion_tesis":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 2,
		"numIndexesAfter" : 3,
		"ok" : 1
	}

Para las búsquedas por autor aplico un índice compuesto de nombre y apellidos.


	db.investigadores.ensureIndex({"nombre":1,"apellidos":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 1,
		"numIndexesAfter" : 2,
		"ok" : 1
	}


**Distribución de los datos**

Dado que las consultas se harían siempre sobre periodos de tiempo, la distribución más eficiente sería sharding basado en rangos.



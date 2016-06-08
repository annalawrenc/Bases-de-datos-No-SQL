

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

Aparece confirmaci�n: 

	WriteResult({ "nInserted" : 1 })

Puedo ver todos los items con:
	
	db.items.find().pretty()


# **Primer ejercicio**

**Actualizar la colecci�n ��tems� para hacer una contra-oferta al primer producto disponible que est� etiquetado como �tel�fono m�vil� y que haya sido puesto en venta con posterioridad al 1/1/2014.**

Declaro una variable de fecha inicial. A continuaci�n a�ado un subdocumento al primer documento que tenga etiqueta "telefono movil" y fecha mayor que la fecha inicial.
En el subdocumento informo email, pasword, la oferta de 45 euros y su fecha.

	var start = new Date(2014, 1, 1)

	db.items.update({tags:"telefono movil","fecha": {$gt: start}},  
	{$addToSet: {  contraoferta: {email: "ggomez@gmail.com", psw: "ggomez",  oferta: 45, fecha: new Date("2016, 5, 9")}}})

Aparece confirmaci�n:

	WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 0 })


# **Segundo ejercicio**

**Actualizar la colecci�n ��tems� para modificar el estado de todos los productos puestos en venta antes de 1/1/2012 y cuyo estado sea disponible. El nuevo estado pasar� a ser descatalogado.**

Declaro una veriable de fecha l�mite. A continuaci�n actualizo estado a "descatalogado" de todos los documentos (por eso uso $multi=true) cuyo estado es "disponible" y fecha es menor que la fecha l�mite.  

	var start = new Date(2012, 1, 1)

	db.items.update({estado:"disponible","fecha": {$lt: start}},
	{$set:{estado:"descatalogado"}},
	{$multi:true})

No se han encontrado productos tan antiguos:

	WriteResult({ "nMatched" : 0, "nUpserted" : 0, "nModified" : 0 })


#**Tercer ejercicio**

**Recuperar la descripci�n y precio de todos los productos etiquetados como �entretenimiento�, cuyo estado sea disponible y que est�n en venta en un punto cercano al nuestro (+- 1000 mts.)**

Busco con comando find los items etiquetados como "entretenimiento", con estado "disponible" y cuya localizaci�n est� cerca (max mil metros) de las coordenadas espec�ficas.
Selecciono descripci�n y precio y elimino el id de la selecci�n. Al final pongo .pretty() para que la salida tenga formato.

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

**Averiguar el n�mero de �tems disponibles que est�n etiquetados como �tel�fono m�vil� y que cuesten menos de 60�.**

Busco los items etiquetados como "telefono movil", estado "disponible" y precio menos que 60. Solo selecciono ID porque quiero unicamente contar con la funci�n count.

	db.items.find(
	{tags: "telefono movil", precio:{$lt:60}, estado: "disponible"},
	{_id:1}).count()

Tambi�n puedo escribir la secuencia de otra forma:

	db.items.count( 
	{tags: "telefono movil", precio:{$lt:60}, estado: "disponible"}, 
	{_id:1}
	)

Encuentro dos:

	2

(esto es porque hab�a introducido dos telefonos: Iphone5 e Ipnone6).

#**Quinto ejercicio**

**Eliminar los registros cuyo estado sea �vendido� y que hubieran sido puestos a la venta antes del 1/1/2012.**

Declaro una variable para la fecha limite y a continuaci�n 

	var start = new Date(2012, 1, 1)

	db.items.remove(
	{estado:"vendido","fecha": {$lt: start}},
	{$multi:true})

No se han encontrado ningunos items tan viejos:

	WriteResult({ "nRemoved" : 0 })

#**Actividad2**

Dado que la base de datos debe servir para consultar informaci�n relacionada con las tesis y con los investigadores voy a dise�ar dos tipos de colecciones: Tesis e Investigadores.

En el agregado **Tesis** se econtrar� informaci�n basica sobre la misma que refleje el contenido y sirva para facilitar la b�squeda:
el t�tulo de la tesis, su autor, palabras clave, codificaci�n UNESCO, instiruci�n y fecha.

En el agregado **Investigadores** estar� la informaci�n sobre los autores y directores.
Asumo que la misma persona podr�a ser autor o director en distintos periodos de tiempo.
Por ello dise�o una colecci�n que permita identificar el rol (autor/director) y, en consecuencia, contenga informaci�n adicional sobre la instituci�n y fecha en la que se desempa�a el rol y la tesis (presentada o dirigida).
Si el investigador de la colecci�n es un autor, se podr� encontrar informaci�n adicional sobre el director de la tesis.
Si el investigador es director, en este caso la informaci�n adicional dar� detalles sobre el autor de la tesis dirigida.

El siguiente esquema refleja el **dise�o en forma gr�fica:**
 
![Doctorados] (https://github.com/annalawrenc/Mongo/blob/master/screens/DisenoMongoTesis.jpg?)

**Implementaci�n**

Creo una base de datos "Doctorados"

	use Doctorados

	switched to db Doctorados

Inserto un ejemplo del documento tipo "Tesis":

	db.tesis.insert({
	autor:{nombre:"Mario", 
	apellidos: "Lopez"},
	titulo:"Elaboraci�n de helados de fresa",
	clave:["helados","fresas"],
	clasificacion_unesco: {clave_un_6dig:"123456", etiqueta_un:"alimentaci�n"},
	fecha_tesis: new Date ("2013, 10, 2"),
	institucion_tesis:"Universidad de Alcal�"
	})

	WriteResult({ "nInserted" : 1 })

Inserto un ejemplo del documento tipo "Investigador". 
En este caso el investigaror es autor de a tesis y aporto informaci�n adicional sobre el director.

	db.investigadores.insert({
	nombre:"Mario", 
	apellidos: "Lopez",
	rol: {autor_director:"autor", institucion_investigador: "Universidad de Alcal�", fecha_rol: new Date ("2013, 10, 2") },
	tesis: {titulo:"Elaboraci�n de helados de fresa", etiqueta_un:"alimentaci�n", 
	director:{nombre: "Hugo",apellidos: "Martinez"},institucion_tesis: "Universidad de Alcal�"}
	})

	WriteResult({ "nInserted" : 1 })

Inserto otro ejemplo del documento tipo "Investigador". 
En este caso el investigaror es director y aporto informaci�n adicional sobre el autor de la tesis.

	db.investigadores.insert({
	nombre:"Hugo", 
	apellidos: "Martinez",
	rol: {autor_director:"director", institucion_investigador: "Universidad Complutense", fecha_rol: new Date ("2013, 10, 2") },
	tesis: {titulo:"Elaboraci�n de helados de fresa", etiqueta_un:"alimentaci�n", 
	autor:{nombre: "Mario",apellidos: "Lopez"},institucion_tesis: "Universidad de Alcal�"}
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
		"titulo" : "Elaboraci�n de helados de fresa",
		"clave" : [
			"helados",
			"fresas"
		],
		"clasificacion_unesco" : {
			"clave_un_6dig" : "123456",
			"etiqueta_un" : "alimentaci�n"
		},
		"fecha_tesis" : ISODate("2013-10-01T22:00:00Z"),
		"institucion_tesis" : "Universidad de Alcal�"
	}


	db.investigadores.find().pretty()
	{
		"_id" : ObjectId("575753662d9d3a4ca7c173c0"),
		"nombre" : "Mario",
		"apellidos" : "Lopez",
		"rol" : {
			"autor_director" : "autor",
			"institucion_investigador" : "Universidad de Alcal�",
			"fecha_rol" : ISODate("2013-10-01T22:00:00Z")
		},
		"tesis" : {
			"titulo" : "Elaboraci�n de helados de fresa",
			"etiqueta_un" : "alimentaci�n",
			"director" : {
				"nombre" : "Hugo",
				"apellidos" : "Martinez"
			},
			"institucion_tesis" : "Universidad de Alcal�"
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
			"titulo" : "Elaboraci�n de helados de fresa",
			"etiqueta_un" : "alimentaci�n",
			"autor" : {
				"nombre" : "Mario",
				"apellidos" : "Lopez"
			},
			"institucion_tesis" : "Universidad de Alcal�"
		}
	}


Adicionalemnte voy a crear �ndices seg�n los criterios de b�squeda m�s comunes:

Para las busquedas por area de conocimiento indexo la etiqueta unesco.


	db.tesis.ensureIndex({"clasificacion_unesco.etiqueta_un":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 1,
		"numIndexesAfter" : 2,
		"ok" : 1
	}

Para las busquedas por instituci�n de la tesis indexo nombre de instiruci�n.


	db.tesis.ensureIndex({"institucion_tesis":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 2,
		"numIndexesAfter" : 3,
		"ok" : 1
	}

Para las b�squedas por autor aplico un �ndice compuesto de nombre y apellidos.


	db.investigadores.ensureIndex({"nombre":1,"apellidos":1})

	{
		"createdCollectionAutomatically" : false,
		"numIndexesBefore" : 1,
		"numIndexesAfter" : 2,
		"ok" : 1
	}


**Distribuci�n de los datos**

Dado que las consultas se har�an siempre sobre periodos de tiempo, la distribuci�n m�s eficiente ser�a sharding basado en rangos.



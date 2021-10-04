# HW---6.6.-Troubleshooting

HW - 6.6. Troubleshooting

## Задача 1

Пользователь (разработчик) написал в канал поддержки, 
что у него уже 3 минуты происходит CRUD операция в MongoDB и её нужно прервать.

- напишите список операций, которые вы будете производить для остановки запроса пользователя

Для остановки запроса пользователя в MongoDB можно использовать db.killOp метод:

The db.killOp() method interrupts a running operation at the next interrupt point. db.killOp() identifies the target operation by operation ID.

db.killOp(<opId>)

Для этого, надо

В случае с read операцией:

1. найти ID операции


		use admin
		db.aggregate( [
   		{ $currentOp : { allUsers: true, localOps: true } },
   		{ $match : <filter condition> } // Optional.  Specify the condition to find the op.
         		                          // e.g. { op: "getmore", "command.collection": "someCollection" }
		] )

2. Остановка операции с использованием найденого ID

		db.killOp(<opid of the query to kill>)


В случае с write

 1.1 В пределах сессии

Если операция имеет ассоциацию с сесией на сервере то можно использовать метод killSessions

Сначала ищем lsid (logical session id).

		use admin
		db.aggregate( [
 		  { $currentOp : { allUsers: true, localOps: true } },
 		  { $match : <filter condition> } // Optional.  Specify the condition to find the op.
      		                              // e.g. { "op" : "update", "ns": "mydb.someCollection" }
		] )

Потом терминируем сессию

	db.adminCommand( { killSessions: [
  	 { "id" : UUID("80e48c5a-f7fb-4541-8ac0-9e3a1ed224a4"), "uid" : BinData(0,"47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=") }
	] } )

	1.2 Вне сессии


Ищем ID операции

		use admin
		db.aggregate( [
 		  { $currentOp : { allUsers: true } },
 		  { $match : <filter condition> } // Optional.  Specify the condition to find the op.
		] )

метод вернёт ID операции с привязкой к шарду.

Терминируем операцию на шардах

		db.killOp("shardB:79014");
		db.killOp("shardA:100813");


Информация из раздела

https://docs.mongodb.com/manual/tutorial/terminate-running-operations/
--------------

- предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

Для тюнинга операций можно предложить следующий порядок действий:

1. Посмотреть какой запрос долгий

	db.currentOps{{"secs_running":{$gte:5}}}

2. Посмотреть его explain

	.explain("executionStats")
3. На основе explain модернизировать запрос.


Дополнительно в Mongo DB можно настроить терминирование долгих операций:

The maxTimeMS() method sets a time limit for an operation. When the operation reaches the specified time limit, 
MongoDB interrupts the operation at the next interrupt point.

Используя метод maxTimeMS()

		db.location.find( { "town": { "$regex": "(Pine Lumber)",
  		                            "$options": 'i' } } ).maxTimeMS(30)

Плюс можно настроить терминирование определённых команд, в случае превышения лимита выполнения при запуске

		db.runCommand( { distinct: "collection",
       		          key: "city",
       		          maxTimeMS: 45 } )

И третье настроить мониторинг и отслеживать время выполнения операций с настройкой соответствующих тригеров.

## Задача 2.




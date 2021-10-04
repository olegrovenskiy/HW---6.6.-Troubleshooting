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

При масштабировании сервиса до N реплик вы увидели, что:

сначала рост отношения записанных значений к истекшим
Redis блокирует операции записи
Как вы думаете, в чем может быть проблема?

Скорее всего при каком то увеличении числа реплик мы попали под ограничение intrinsic latency,
которые вызваны:

1. Средой виртуализации
2. Сетевыми задержками
3. Аппаратными задержками
4. Задержками операционной системы


По блокировке операций записи - Redis является однопоточным, он скорее всего ничего не блокирует, 
но он не обрабатывает никакие другие команды,пока предыдущая команда не завершится. 

## Задача 3


Задача 3

Появились ошибки вида:

InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
Как вы думаете, почему это начало происходить и как локализовать проблему?

Какие пути решения данной проблемы вы можете предложить?


Согласно документу 
https://dev.mysql.com/doc/refman/8.0/en/error-lost-connection.html
могут быть следующин причины:

1. Сетевыею вряд ли наш случай, т.к. есть комментарий При росте количества записей, в таблицах базы

2. Причина из за большого количества строк, отправляемых в запросе

Sometimes the “during query” form happens when millions of rows are being sent as part of one or more queries. 
If you know that this is happening, you should try increasing net_read_timeout from its default of 30 seconds 
to 60 seconds or longer, sufficient for the data transfer to complete.

Это подходит под наш случай, и способ решения увеличить net_read_timeout from its default of 30 seconds 
to 60 seconds or longer

3. Реже это может произойти, когда клиент пытается выполнить первоначальное подключение к серверу. - 
тоже не наш случай.

Для локализации проблемы:

 - проверить сетевые соединения, что бы исключить проблемы с сетью
 - провести мониторинг параметров сервера (CPU, Memmory), возможно не хватает ресурсов. 
   Если это так то решение - увеличить ресурсы.
 - возможно есть запосы которые сильно грузят базу, проверить есть ли замедления пользовательских запросов
   через slow_log. если есть, то через explain изучить причины и оптимизировать. 
   Рестарт процесса mysql может помочь.

Если ни одна из причин не подходит, то возможно BLOB больше чем установлен max_allowed_packet, 
что можент создавать проблемы некоторым клиентам. Решение увеличить  max_allowed_packet. Этот вариант 
вряд ли наш тоже, т.к. были бы тогда ещё ошибки ER_NET_PACKET_TOO_LARGE.

Предложенный вариант временно решает проблему, не не устраняет первопричину - рост количества записей, в таблицах базы.

Системное решение:

 - уменьшить объем данных в базе данных, удалив старую информацию
 - разделить таблицы баз данных на более мелкие - Партиционирование (partitioning)
 - вертикальное разделение таблицы
 - или сжать таблицы.

## Задача 4
	


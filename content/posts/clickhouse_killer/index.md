+++
title = 'Убийца кластера Clickhouse'
date = 2024-01-24T20:37:08+03:00
draft = false
+++

Описанная далее проблема случилась уже давно, примерно давно и было найдено её решение. Всё затягивал я и затягивал с выкладыванием этого материала, но довольно. Представляю вашему вниманию детектив об **убийце кластера Clickhouse**.

## Входные данные

Начнем с входных данных. Кластер на 3 машины настроен и вполне работает. Конфиги, которые я использовал, если вам интересно, можете обнаружить в одном из предыдущих постов.

"Почему я думаю, что кластер работает?" - может возникнуть вопрос. "Потому что я угрожал его семье" - скажу я. Шутка. Я так считал и не отказываюсь от этого, у меня, в конце концов, есть основания: сервера отвечают корректно на команду **stat**.

## Тот самый убийца кластера

Перезагружаем какой- нибудь сервер кластера и... После перезагрузки сервера кластеру становится плохо. Кластер больше не может прийти к консенсусу.

```
2023.12.05 09:13:48.707111 [ 11591 ] {2c49171b-944c-4bf3-a7f7-78644aa99a65} <Err
or> executeQuery: Code: 999. Coordination::Exception: Connection loss. (KEEPER_E
XCEPTION) (version xxx) (from 0.0.0.0:0) (in query: /* ddl_entry=query-000
0000003 */ CREATE TABLE IF NOT EXISTS table_name.bd_name, Stack trace (when copying this messag
e, always include the lines below):

0. Poco::Exception::Exception(String const&, int) @ 0x1494bd7a in /nix/store/lx6
scx072nzfbybn9g45l9h8mqcx3b7r-clickhouse-xxx/bin/clickhouse
1. DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0xb326a
43 in /nix/store/lxbscx213nzfbybn9g45lfg8mqcx3b7r-clickhouse-xxx

...

2023.12.05 09:13:48.768152 [ 11591 ] {2c49171b-944c-4bf3-a7f7-78644aa99a65} <Err
or> DDLWorker: Query /* ddl_entry=query-0000000003 */ CREATE TABLE IF NOT EXISTS
 TABLE_NAME UUID '1981e0ac-244c-4b4c-826b-c80471e9c012' ( ... ) ENGINE = ReplicatedMergeTree
 PARTITION BY (...) ORDER BY ... wasn't finished successfully: Code: 999. Coordination::Exception: Connection
 loss. (KEEPER_EXCEPTION), Stack trace (when copying this message, always includ
e the lines below):

0. Poco::Exception::Exception(String const&, int) @ 0x1494bd7a in /nix/store/lxbscx213nzfbybn9g45lfg8mqcx3b7r-clickhouse-xxx/bin/clickhouse
1. DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0xb326a
43 in /nix/store/lxbscx213nzfbybn9g45lfg8mqcx3b7r-clickhouse-xxx/bin/click
house
2. Coordination::Exception::Exception(String const&, Coordination::Error, int) @
 0x12359582 in /nix/store/lxbscx213nzfbybn9g45lfg8mqcx3b7r-clickhouse-xxx/
bin/clickhouse

...
```

*ip- адреса, доменные имена, порты и названия полей в таблице я намеренно скрываю или прячу*

## Попытка осознать

Итак, всё, кластер разъехался, а в него даже данных никто не слал. Был **лишь перезапущен сервер** у нас ничего не работает. Только что успешно все работало и биты информации бегали от сервера к серверу, а тут оп и приехали...

В ответ на stat кластер говорит следующее:

``` bash
root@localhost:~# echo stat | nc 172.16.128.251 9181 ; echo ; echo stat | nc 172.16.128.252 9181 ; echo ; echo stat | nc 172.16.128.253 9181
```


``` bash
This instance is not currently serving requests

ClickHouse Keeper version: vxxx
Clients:
 ip1:40815(recved=0,sent=0)
 127.0.0.1:57636(recved=289966,sent=289985)

Latency min/avg/max: 0/0/65
Received: 289966
Sent: 289985
Connections: 1
Outstanding: 0
Zxid: 14798
Mode: leader
Node count: 4968

ClickHouse Keeper version: vxxx
Clients:
 ip1:41747(recved=0,sent=0)
 127.0.0.1:37056(recved=234648,sent=234668)

Latency min/avg/max: 0/0/41
Received: 234648
Sent: 234668
Connections: 1
Outstanding: 0
Zxid: 14798
Mode: follower
Node count: 4968
```

А вот ответ на lgif можно увидеть следующую картину:

``` bash
root@localhost:~# echo lgif | nc ip1 9181 ; echo ; echo lgif | nc ip2 9181 ; echo ; echo lgif | nc ip3 9181
first_log_idx	1
first_log_term	1
last_log_idx	14464
last_log_term	5
last_committed_log_idx	0
leader_committed_log_idx	0
target_committed_log_idx	0
last_snapshot_idx	0

first_log_idx	1
first_log_term	1
last_log_idx	14998
last_log_term	55
last_committed_log_idx	14998
leader_committed_log_idx	14998
target_committed_log_idx	14998
last_snapshot_idx	9678

first_log_idx	1
first_log_term	1
last_log_idx	14998
last_log_term	55
last_committed_log_idx	14998
leader_committed_log_idx	14998
target_committed_log_idx	14998
last_snapshot_idx	0
```

На данном этапе в той ситуации можно было проводить вскрытие жертвы. Уже никто не мог её спасти, кластер *разъехался* окончательно. На самом деле спасти его как бы еще можно было. Но [с применением средства после потери кворума](https://clickhouse.com/docs/ru/operations/clickhouse-keeper#%D0%B2%D0%BE%D1%81%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BF%D0%BE%D1%81%D0%BB%D0%B5-%D0%BF%D0%BE%D1%82%D0%B5%D1%80%D0%B8-%D0%BA%D0%B2%D0%BE%D1%80%D1%83%D0%BC%D0%B0) (сброса прогресса на лидере и выбор нового лидера с последующей пересинхронизацией).

## Следственный эксперимент

Откатываем все сервера и повторяем процедуру, заранее удостоверяемся в том, что команда stat верно обрабатывается серверами.

``` bash
root@localhost:~# echo stat | nc ip1 9181 ; echo ; echo stat | nc ip2 9181 ; echo ; echo stat | nc ip3 9181

ClickHouse Keeper version: vxxx
Clients:
 ip1:38371(recved=0,sent=0)
 127.0.0.1:44202(recved=10,sent=10)

Latency min/avg/max: 0/1/4
Received: 10
Sent: 10
Connections: 1
Outstanding: 0
Zxid: 21
Mode: follower
Node count: 7

ClickHouse Keeper version: vxxx
Clients:
 ip1:36929(recved=0,sent=0)
 127.0.0.1:57856(recved=10,sent=10)

Latency min/avg/max: 0/21/135
Received: 10
Sent: 10
Connections: 1
Outstanding: 0
Zxid: 21
Mode: leader
Node count: 7

ClickHouse Keeper version: vxxx
Clients:
 ip1:33281(recved=0,sent=0)
 127.0.0.1:37236(recved=10,sent=10)

Latency min/avg/max: 0/22/135
Received: 10
Sent: 10
Connections: 1
Outstanding: 0
Zxid: 21
Mode: follower
Node count: 7
```

Все ок с кластером. Далее интересное: при **кратковременном выключении одного из улов, кластер может собраться вновь**. Эмпирическим путем было установлено, что для этого разница между наименьшим и наибольшим в кластере значениями last_log_idx не должна превышать примерно 30.

```
root@localhost:~# echo lgif | nc ip1 9181 ; echo ; echo lgif | nc ip2 9181 ; echo ; echo lgif | nc ip3 9181

first_log_idx	1
first_log_term	1
last_log_idx	88
last_log_term	1
last_committed_log_idx	88
leader_committed_log_idx	88
target_committed_log_idx	88
last_snapshot_idx	0

first_log_idx	1
first_log_term	1
last_log_idx	88
last_log_term	1
last_committed_log_idx	88
leader_committed_log_idx	88
target_committed_log_idx	88
last_snapshot_idx	0

first_log_idx	1
first_log_term	1
last_log_idx	88
last_log_term	1
last_committed_log_idx	88
leader_committed_log_idx	88
target_committed_log_idx	88
last_snapshot_idx	0
```

А вот записать в такой кластер не получится, даже создание таблицы оборачивается проблемами:

```
CREATE TABLE IF NOT EXISTS  testdb.testtable ON CLUSTER test_c
(
    id Int8,
) ENGINE = ReplicatedMergeTree()
ORDER BY id;
```

Но в ответ в веб-морде вернется такое исключение:
```
Code: 999. Coordination::Exception: Connection loss, path: /какой-то_путь/task_queue/ddl/query-0000000000: While executing DDLQueryStatus. (KEEPER_EXCEPTION) (version 23.3.13.1)
```

Но вид ошибки может быть и другим. Это зависит от того, к лидеру или к последователю отправляется запрос на создание таблицы в первую очередь.

```
{
	"meta":
	[
		{
			"name": "host",
			"type": "String"
		},
		{
			"name": "port",
			"type": "UInt16"
		},
		{
			"name": "status",
			"type": "Int64"
		},
		{
			"name": "error",
			"type": "String"
		},
		{
			"name": "num_hosts_remaining",
			"type": "UInt64"
		},
		{
			"name": "num_hosts_active",
			"type": "UInt64"
		}
	],

	"data":
	[
		["ip1", 9000, "999", "Code: 999. Coordination::Exception: Connection loss. (KEEPER_EXCEPTION) (version xxx)", "2", "2"],
		["ip2", 9000, "999", "Code: 999. Coordination::Exception: Operation timeout, path: \/clickhouse\/tables. (KEEPER_EXCEPTION) (version xxx)", "1", "1"],
		["ip3", 9000, "999", "Code: 999. Coordination::Exception: Operation timeout. (KEEPER_EXCEPTION) (version xxx)", "0", "0"]Code: 999. DB::Exception: There was an error on [ip3:port]: Code: 999. Coordination::Exception: Connection loss. (KEEPER_EXCEPTION) (version xxx). (KEEPER_EXCEPTION) (version xxx)
    ]
```

## Ничего не понятно, но...

В какой- то момент, вводя случайные команды в терминале одного из серверов ,попадаю я на список сетевых интерфейсов и вижу, что подозрительно много пакетов отправились в DROP.

Казалось бы, из- за чего их так много?

Сделаю небольшую оговорку, а затем раскрою все карты. Ноды кластера представляли собой отдельные виртуальные машины на разных физических серверах.

Вот, что я увидел на виртуальной машине: два интерфейса с MTU 9000.

```
root@localhost:~# ifconfig | grep MTU
          UP BROADCAST RUNNING MULTICAST  MTU:9000  Metric:1
          UP BROADCAST RUNNING MULTICAST  MTU:9000  Metric:1
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
```

А вот уже данные с физической машины, на которой хостилась эта виртуалка. Два последних интерфейса- те, которые прокинуты в виртуальную машину. Один из оставшихся сетевых интерфейсов соединяет физический сервер с остальными. На нём MTU равен 1500.

``` bash
root@computer01:~# ifconfig | grep MTU
          ...
          UP BROADCAST SLAVE MULTICAST  MTU:1500  Metric:1
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          UP BROADCAST RUNNING MULTICAST  MTU:9000  Metric:1
          UP BROADCAST RUNNING MULTICAST  MTU:9000  Metric:1
```

Вот она и отгадка загадки. Из- за неверного MTU, в ситуации, когда данных становилось слишком много, пакеты с данными Clickhouse Keeper отбрасывались на интерфейсе. В результате чего соединения нод отключались по таймауту, не дождавшись ответа.

## Вот и всё. Сделали выводы, запомнили на будущее.

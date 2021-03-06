﻿#Не закончено!

Проектик **stress**, может показать, как работает библиотека в тех или иных условиях. Адрес примера: https://github.com/newenclave/vtrc/tree/master/examples/stress

Как и во всех остальных примерах, код разделен на 3 части: 

 * client — клиентская часть 
 * protocol — описание протокола (общая часть, используемая и клиентом, и сервером)
 * server — серверная часть 

Доступные опции серверной части: **./stress_server --help**

    -? [ --help ]              help message
    -s [ --server ] arg        endpoint name; <tcp address>:<port> or <pipe/file 
                               name>
    -i [ --io-pool-size ] arg  threads for io operations; default = 1
    -r [ --rpc-pool-size ] arg threads for rpc calls; default = 1
    -t [ --tcp-nodelay ]       set TCP_NODELAY flag for tcp sockets
    -o [ --only-pool ]         use io pool for io operations and rpc calls
    -M [ --message-size ] arg  maximum message size for protocol
    -B [ --read-size ] arg     buffer size for receiving data
    -S [ --stack-size ] arg    maximum recursion depth for both side

тут: 

* help — вывод помощи

* **server** — список открываемых серверных точек, имена точек задаются в формате ipaddress:port, либо local_path_name, где  local_path_name может быть именем *Named pipe* в случае системы Windows или именем *Unix domain socket* в случае POSIX систем. Точек может быть несколько.

* **io-pool-size** — количество потоков, используемых для обработки системных событийс (чтение сокета или пайпа, прием новых клиентов). Может принимать значение от 1 и больше.

* **rpc-pool-size** — количество потоков, назначенных на выполнение процедур. Все удаленные вызовы будут выполнятся в этих потоках. Может принимать значение от 1 и больше.

* **only-pool** — параметр, который говорит серверу обрабатывать и системные события, и удаленные вызовы на одном и том же диспетчере, параметр **io-pool-size**, в этом случае задает количество потоков для этого диспетчера, параметр **rpc-pool-size** игнорируется. Запуск сервера с одним только ключом **--only-pool** запустит сервер только с 1 потоком (это будет главный поток функции main) и все все будет обрабатываться в нём.

* **message-size** — задает максимальный размер сообщения с запросом, либо ответом. Превышение этого размера является грубым нарушение протокола и соединение незамедлительно закрывается. По-умолчанию это значение равно 65536 байт. Максимальное значение — 64 мегабайта.

* **read-size** – размер буфера приема. Именно столько байт сможет за один прием вычитать сервер из канала клиента. 4096 байт по-умолчанию.

* **stack-size** — задает максимально допустимую глубину вызовов (см description.md)

* **tcp-nodelay** — устанавливает флаг TCP_NODELAY для tcp соединений. Этот флаг отключает действие *алгоритма Нейгла* (http://tools.ietf.org/html/rfc896). Действие этого алгоритма очень хорошо просматривается при выполнении stress теста с событиями (см. ниже запрос событий у сервера клиентом), используя tcp соединения.

Пример: 
Запуск сервера:

    ./stress_server --server=0.0.0.0:55555 --server=:::55556 --server=/home/sandbox/stress.sock -o

    Starting listener at '0.0.0.0:55555'...Ok
    Starting listener at ':::55556'...Ok
    Starting listener at '/home/sandbox/stress.sock'...Ok


Запустит сервер с 3 точками и будет использовать поток функции main для всей работы. В такой схеме без проблем работают вызовы, которые не требуют долгой обработки на стороне сервера. Всё выполняется последовательно. 

При 

    ./stress_server --server=0.0.0.0:55555 --server=:::55556 --server=/home/sandbox/stress.sock -i2 -r4

Запустится сервер с теми же 3 точками, но для обработки системных событий будет использовано 2 потока, а для исполнения вызовов — 4. При в роли одного из потоков RPC диспетчера будет выступать поток функции main. Итого: основной поток + 5 дополнительных.



##Client

Доступные опции клиентской части

    Allowed options:
        -? [ --help ]              help message
        -s [ --server ] arg        server name; <tcp address>:<port> or <pipe/file 
                                   name>
        -i [ --io-pool-size ] arg  threads for io operations; default = 1
        -r [ --rpc-pool-size ] arg threads for rpc calls; default = 1
        -t [ --tcp-nodelay ]       set TCP_NODELAY flag for tcp sockets
        -p [ --ping ] arg          make ping [arg] time
        -c [ --gen-callbacks ] arg ask server for generate [arg] callbacks
        -e [ --gen-events ] arg    ask server for generate [arg] events
        -R [ --recursive ] arg     make recursion calls
        -l [ --payload ] arg       payload in bytes for commands such as ping; 
                                   default = 64

Тут: 

**server** – адрес сервера, имеет тот же формат, что адрес точки, используемой при запуске сервера (127.0.0.1:55555, ::1:55556, /home/sandbox/stress.sock, etc)

**io-pool-size**,  **rpc-pool-size**, **tcp-nodelay** – отвечают за то же, что аналогичные опции серверной стороны. (todo: придумать пример, показывающий влияние tcp-nodelay для клиентской стороны...посылка множества событий на сторону сервера, например).

Далее:

##payload

Общая настройка для остальных методов. Payload это симуляция полезной нагрузки для удаленных вызовов. По-умолчанию принимает значение 64 (уж не знаю почему =) ). В сообщения добавляется ```[arg]``` байт данных. 

##ping

Принимает количество запросов, которые необходимо выполнить. Пауза между вызовами — 1 секунда.
Для примера используется сообщение, описанное в файле протокола (stress.proto) 

```proto

message ping_req {
    optional bytes payload = 1;
}
message ping_res { } // пустой ответ.

```

а сам вызов описан в секции 

```proto
service stress_service { 
    ....
    rpc ping (ping_req) returns (ping_res);
    ....
}
```

**ping_req** - сообщение-запрос. В качестве payload имеет размер, указанный в опции payload командной строки.

**ping_req** пустой и не несет никакой полезной нагрузки. Описан только для того, чтобы быть использованным в описании вызова.

На машине с адресом 10.0.0.1 есть stress_server, слушающий порт 55555, тогда, выполнив ```./stress_client -s 10.0.0.1:55555 -p 5``` получим 

```console
Creating client ... Ok
Connecting to 10.0.0.1:55555...connect...ready...Ok
Start pinging...
Send ping with 64 bytes as payload...ok; 292 microseconds
Send ping with 64 bytes as payload...ok; 353 microseconds
Send ping with 64 bytes as payload...ok; 494 microseconds
Send ping with 64 bytes as payload...ok; 590 microseconds
Send ping with 64 bytes as payload...ok; 619 microseconds
Stopped
```

Увеличив  payload до, например, 44000 ```./stress_client -s 10.0.0.1:55555 -p 5 --payload=44000```, получим

```console
Creating client ... Ok
Connecting to 10.0.0.1:55555...connect...ready...Ok
Start pinging...
Send ping with 44000 bytes as payload...ok; 837 microseconds
Send ping with 44000 bytes as payload...ok; 1075 microseconds
Send ping with 44000 bytes as payload...ok; 926 microseconds
Send ping with 44000 bytes as payload...ok; 1097 microseconds
Send ping with 44000 bytes as payload...ok; 1225 microseconds
Stopped
```

То есть время, потраченное для отправки  44000 байт на сторону сервера, выполнение вызова на стороне сервера и прием ответа, получилось в районе 1 миллисекунды. 


Если же мы зададим серверу маленький **read-size**, например равный 1 ```./stress_server -s 0.0.0.0:55555 --read-size=1```, то получим 

```console
Creating client ... Ok
Connecting to 10.0.0.1:55555...connect...ready...Ok
Start pinging...
Send ping with 44000 bytes as payload...ok; 52349 microseconds
Send ping with 44000 bytes as payload...ok; 53835 microseconds
Send ping with 44000 bytes as payload...ok; 47642 microseconds
Send ping with 44000 bytes as payload...ok; 59027 microseconds
Send ping with 44000 bytes as payload...ok; 54667 microseconds
Stopped
```

50-60 миллисекунд.















Отложить 

```bash
#!/bin/bash

n=0
while [ $n -lt 2000 ];
do
    n=$[$n+1];
    ./stress_client -s ::1:55555 -f 50 --payload=44000 &

done
```


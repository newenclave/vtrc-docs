## HELLO


Простой пример использования библиотечки. События и колбеки.
Тут будет описана только та часть, которая связана непосредственно с событиями. Для информации обо всех остальных деталях смотреть [example-howto](https://github.com/newenclave/vtrc-docs/blob/master/ru/example-hello.md)

##Protocol 

Протокол декларирован в [proto-файле](https://github.com/newenclave/vtrc/blob/master/examples/hello-events/protocol/hello-events.proto). 

```protobuf

package howto;
option cc_generic_services = true;

message request_message {
    optional string name = 1;
}

message response_message {
    optional string hello = 1;
}

service hello_events_service {
    rpc generate_events( request_message ) returns ( response_message );
}

message event_req { }
message event_res {
    optional string hello_from_client = 1;
}

service hello_events {
    rpc hello_event(event_req) returns (event_res);
    rpc hello_callback(event_req) returns (event_res);
}
```

Основное отличие от примера без событий — это наличие второго сервиса ```hello_events```, в котором есть 2 вызова. ```hello_event``` и ```hello_callback```. Первый будет работать в качестве события, второй в качестве колбека, который может вернуть результат.

##Server 

Прежде всего процесс сервера будет использовать несколько потоков для работы (в нашем случае 2). Это все потому что вызов колбека блокирующий, и серверу нужен еще один поток, чтоб получать новые соединения, новые данные от клиентов.

Поэтому в заголовках вместо 

```cpp
#include "vtrc-common/vtrc-thread-pool.h"

```
будет включен 

```cpp
#include "vtrc-common/vtrc-pool-pair.h"
```
Эта пара позволяет разделить обработку системных событий и выполнение удалённых вызовов. 

Кроме того подключен хедер для работы с оберткой Stub-классов

```cpp
#include "vtrc-common/vtrc-stub-wrapper.h"
```

Теперь вместо создания одного потока, создается пара.

```cpp
...
    common::pool_pair pp( 0, 1 );
    hello_application app( pp );
...

```

тут создается всего один поток, а первый ноль указывает на то, что нужно создать io_service, но пока не запускать обработку для него.



##Client















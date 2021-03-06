﻿## HELLO


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

Доступ к IO пулу у `common::pool_pair` возможен через `get_io_pool( )`. 

```cpp
        pp.get_io_pool( ).attach( );
```
Как и в примере `hello`, запустит обработку системных событий в потоке вызова `main( )`.


Далее работа идентична примеру [hello](https://github.com/newenclave/vtrc-docs/blob/master/ru/example-hello.md), однако теперь в вызове главного сервиса `generate_events` создается 2 разных канала и через эти 2 канала делаются 2 разных вызова.

####Канал первый, вызов события.

```cpp
using   server::channels::unicast::create_callback_channel;
using   server::channels::unicast::create_event_channel;
{ // do event. send and dont wait response
    common::rpc_channel *ec =
            create_event_channel( cl_->shared_from_this( ) );
    common::stub_wrapper<stub_type> event( ec );
    event.call( &stub_type::hello_event );
}
```
2 строчки `using` просто для удобства. Вызовы для создания каналов находятся в `namespace vtrc::server::channels`.

Далее создается канал событий `create_event_channel`. Этот канал "говорит" другой стороне, что нужно выполнить вызов в любом свободном от ожидания потоке. Вызов возвращает управление сразу и не ждет ответа, другая сторона, ответ и не отправляет.  
    
    Однако канал событий можно заставить ждать ответ. 
    Второй параметр вызова create_event_channel, названный disable_wait, отвечает за ожидание. 
    По-умолчанию установлен в true. При значении false, 
    вызовы сделанные через такой канал будут ждать результата с другой стороны.

####Канал второй, сделаем callback.

С колбеком происходит все немного по другому. Во-первых по-умолчанию колбек выполняется с ожиданием результата (`disable_wait` установлен в `false`), во-вторых колбек выполняется на другой стороне в потоке того самого вызова, из которого этот колбек сделан. В этом случае это поток из которого клиент делает `generate_events`. 

```cpp
{ // do callback. wait response from client.

    common::rpc_channel *cc =
        create_callback_channel( cl_->shared_from_this( ) );

    common::stub_wrapper<stub_type> callback( cc );
    howto::event_res response;

    callback.call_response( &stub_type::hello_callback, &response );

    std::cout << "Client string: "
        << response.hello_from_client( )
        << "\n"
        ;
}

```

Тут создается канал вызовом `create_callback_channel` и вызов делается уже с ожиданием. После вызова выводится строка (`response.hello_from_client( )`), которую написал серверу клиент.


##Client

Как и в случае сервера вместо `thread_pool` будет использоваться `pool_pair`.


Кроме того, для клиента теперь нужно создать класс-наследник от класса `howto::hello_events` и устанавливать его в качестве обработчика.

```cpp
class hello_event_impl: public howto::hello_events {
... ... 
};

```

Установим обработчик для вызовов со стороны сервера 

```cpp
    cl->assign_rpc_handler( vtrc::make_shared<hello_event_impl>( ) );
```

Клиент, при старте, выводит id потока функции `main( )`

```cpp
    std::cout << "main( ) thread id is: 0x"
              << std::hex
              << vtrc::this_thread::get_id( )
              << std::dec
              << "\n";
```
После этого делается вызов `generate_events` 

```cpp
        hello.call( &stub_type::generate_events );
```

В этом вызове сервер делает вызов события и колбека. Обработка этих вызовов описана в `hello_event_impl`.

Обработка event 

```cpp
    void hello_event(::google::protobuf::RpcController* /*controller*/,
                 const ::howto::event_req*              /*request*/,
                 ::howto::event_res*                    /*response*/,
                 ::google::protobuf::Closure* done)
    {
        common::closure_holder ch( done );
        std::cout << "Event from server; Current thread is: 0x"
                  << std::hex
                  << vtrc::this_thread::get_id( )
                  << std::dec
                  << "\n"
                     ;
    }

```
Просто выводит id текущего потока. Так как это событие, которое исполняется вне контекста `generate_events`, id будет отличатсья от id потока `main( )`.

Второй обработчик опять же выводит подобную строку, и устанавливает строку для сервера, которую сервер потом выведет на консоль.

```cpp

    void hello_callback(::google::protobuf::RpcController* /*controller*/,
                 const ::howto::event_req*                 /*request*/,
                 ::howto::event_res* response,
                 ::google::protobuf::Closure* done)
    {
        common::closure_holder ch( done );
        std::cout << "Callback from server; Current thread is: 0x"
                  << std::hex
                  << vtrc::this_thread::get_id( )
                  << std::dec
                  << "\n"
                     ;
        std::ostringstream oss;
        oss << "Hello there! my thread id is 0x"
            << std::hex
            << vtrc::this_thread::get_id( )
               ;
        response->set_hello_from_client( oss.str( ) );
    }

```
Тут на консоль уже будет выведен тот самый id, который был показан при старте клиента из функции `main( )`. 

##Пример старта и вывода.

Сервер: 
```
[root@virt2real default]# ./hello_events_server 0.0.0.0 12345
```
Клиент: 
```
% ./hello_events_client 192.168.3.1 12345
Connecting...
connect...ready...Ok
main( ) thread id is: 0x7f89591ac780
Make call 'generate_events'...
Event from server; Current thread is: 0x7f8957492700
Callback from server; Current thread is: 0x7f89591ac780
'generate_events' OK
```

Результат на стороне сервера:

```
[root@virt2real default]# ./hello_events_server 0.0.0.0 12345
Client string: Hello there! my thread id is 0x7f89591ac780
```

Как видно из вывода, событие было вызвано в потоке с id `0x7f8957492700`, а вызов колбека был сделан в потоке с id `0x7f89591ac780`, который является потоком функции `main`.

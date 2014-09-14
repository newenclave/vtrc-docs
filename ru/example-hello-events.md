/* UTF8 */

## HELLO


Простой пример использования библиотечки. События и колбеки.
Тут будет описана только та часть, которая связана непосредственно с событиями. Для информации обо всех остальных деталях смотреть [example-howto](https://github.com/newenclave/vtrc-docs/blob/master/ru/example-hello.md)

##Protocol 

Протокол декларирован в proto-файле (https://github.com/newenclave/vtrc/blob/master/examples/hello-events/protocol/hello-events.proto). 

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


##Server 

##Client















/* UTF8 */

#Не закончено

Как построить клиент, сервер ... 

Для начала нужно описание протокола, который будут понимать обе стороны (что очевидно).

##Protocol

Для описания протокола используются proto-файлы, из которых после, посредством Potobuf, генерируются исходники. (Если читатель не знаком с Protocol Buffers, то самое время ознакомится тут https://code.google.com/p/protobuf/; Обычно хватает часа, а может и меньше, чтоб понять что это зачем и как этим что-то описать и получить).

Обычно я использую разделение проекта на 3 части. Client, Server и Protocol.
Часть "Protocol" используют и client, и server, так что это общая часть проекта. 

Proto-файлов может быть несколько, в каждом может быть описан свой сервис и сообщения используемые этим сервисом. 

```protobuf
// пример proto-файла; пусть это будет howto.proto

package howto_example; // имя namespace для C++

/// cc_generic_services говорит протобуферу, 
/// что нужно сгенерировать сервис; 
/// без этого параметра будет сгенерирован исходник, 
/// описывающий только сообщения, но не service
option cc_generic_services = true; 

message request_message { 
    optional string hello = 1;
}

message response_message { 
    optional string hello = 1;
}

service hello_service {
    rpc send_hello( request_message ) returns ( response_message );
}

```

```
Далее по тексту: 
    "сообщение" — сообщение описанное в proto-файле
    "сервис" – класс, реализующий "service", описанный в proto-файле.
```

в этом файле я описал 2 сообщения ```request_message``` и ```response_message```. Первое — сообщение-запрос, которое нужно будет заполнить клиенту перед тем, как отправить на сторону сервера. Второе – ответ и его заполнит уже сервер, когда выполнит вызов клиента и отправит ответ. 

После генерации мы получим 2 файла  howto.pb.h и howto.pb.cc (если наш файл называется howto.proto), которые должны будут включены в проект. Я использую cmake для генерирования и автоматического включения этих файлов в состав проекта, так мне не нужно думать о перегенерации файлов при изменении *.proto. 

Если в протофайле задана опция  cc_generic_services (её можно так же задать в командной строке protoc), то помимо сообщений будут сгенерированы 2 (два) класса для работы с сервисами. Первый класс (назовем его интерфейсом для сервиса) — класс для реализации обработчика запросов. Этот класс носит ровно то имя, которое было задано в proto-файле. То есть в нашем случае это будет класс  hello_service (howto_example ::hello_service, если с именем namespace). Этот класс содержит **виртуальные** методы, которые реализуют методы rpc, описанные сервисе. У нас будет один метод –  send_hello. В С++ он будет выглядеть так (howto.pb.h):

```cpp
class hello_service : public ::google::protobuf::Service {
.........
    virtual void send_hello(::google::protobuf::RpcController* controller,
                       const ::howto_example::request_message* request,
                       ::howto_example::response_message* response,
                       ::google::protobuf::Closure* done);
.........
};

```

Помимо сообщений запроса и ответа, описанных в протофайле protoc добавил еще 2 сущности **controller** и **done**(замыкание), которые являются служебными и используются на стороне как клиента, так и сервера. О них ниже.

Нагенерённые методы не являются чисто-виртуальными, у каждого из них есть реализация в  соответвующем *.pb.cc файле. Реализация не пустая, в ней устанавливается ошибка в controller. Ошибка говорит о том, что данный метод  (send_hello), данного сервиса (hello_service) не реализован. Такая заглушка. Далее эта ошибка передается клиенту. 

```cpp
void hello_service::send_hello(::google::protobuf::RpcController* controller,
                       const ::howto_example::request_message* request,
                       ::howto_example::response_message* response,
                       ::google::protobuf::Closure* done) {
  controller->SetFailed("Method send_hello() not implemented.");
  done->Run();
}
```

Этот класс мы будем использовать на стороне, которая хочет реализовать этот сервис у себя. Это не обязательно должна быть серверная сторона. Это может и сервер, и клиент, и обе стороны одновременно. Использование заключается в наследовании от данного класса и переопределении виртуальных методов. Что нужно еще см. раздел **Server**.

Второй класс, который получается из сервиса, описанного в *.proto – это Stub класс и будет использоваться для доступа к сервису на другой стороне, которая реализует этот сервис. В нашем случае этот класс будет называться hello_service_Stub. Этот класс уже не является интерфейсным, то есть у него нет виртуальных методов. В конструкторе этот класс принимает указатель на RpcChannel, котороый использует для отправки запросов. О  RpcChannel, опять же, ниже. Методы имеют ровно тот же вид, что и в классе-интерфейсе. То есть 


```cpp
class hello_service : public ::google::protobuf::Service {
.........
    void send_hello(::google::protobuf::RpcController* controller,
                    const ::howto_example::request_message* request,
                    ::howto_example::response_message* response,
                    ::google::protobuf::Closure* done);
.........
};

```
Но уже с готовой реализацией.

Итого: Сторона, которая хочет реализовать некий метод rpc, описанный в *.proto файлах, создает класс-наследник от класса-интерфейса и реализовывает нужные методы. 
Сторона, которая хочет получить доступ к методам сервиса первой стороны, должна использовать Stub-класс и вызывать его методы.

Сторона реализующая сервис (не обязательно сервер!):
```cpp 

class  hello_service_impl: public  howto_example::hello_service {
    void send_hello(::google::protobuf::RpcController* controller,
                    const ::howto_example::request_message* request,
                    ::howto_example::response_message* response,
                    ::google::protobuf::Closure* done) override
    {
        std::ostringstream oss;
        oss << "Hello " << request->hello( ) 
            << " from hello_service_impl::send_hello!";
        response->set_hello( oss.str( ) );
        done->Run( ); /// Этот Run отошлет ответ. См ниже.
    }
};

```

Сторона-клиент
```cpp
howto_example::hello_service_Stub stub(channel);
howto_example::request_message  req;
howto_example::response_message res;
req.set_hello( "%USERNAME%" );

/// тут параметры controller и done, (как и запрос, ответ) 
/// могут быть NULL
stub.send_hello( NULL, &req, &res, NULL ); 

std::cout <<  res.hello( ) << std::endl; 
```

результат клиента
```console
Hello %USERNAME% from hello_service_impl::send_hello!
```

Клиент и сервер могут находиться как одном хосте, так и на разных континентах. 

####Опции сервисов и методов.

TODO: написать

##Server

##Client



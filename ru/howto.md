/* UTF8 */

#Не закончено

Protocol: https://github.com/newenclave/vtrc-docs/blob/master/ru/howto.md#protocol

Common: https://github.com/newenclave/vtrc-docs/blob/master/ru/howto.md#common

Server: https://github.com/newenclave/vtrc-docs/blob/master/ru/howto.md#server

Client: https://github.com/newenclave/vtrc-docs/blob/master/ru/howto.md#client

Как построить клиент, сервер ... 

Для начала нужно описание протокола, который будут понимать обе стороны (что очевидно).

##Protocol

> Тут под протоколом понимается не низкий уровень, который бегает по каналу (о нем опишу отдельно), но протокол уровня сообщений и сервисов, которые описываются и используются при работе.

Для описания протокола используются proto-файлы, из которых после, посредством Potobuf, генерируются исходники. (Если читатель не знаком с Protocol Buffers, то самое время ознакомится тут https://code.google.com/p/protobuf/; Обычно хватает часа, а может и меньше, чтоб понять что это зачем и как этим что-то описать и получить).

Обычно я использую разделение проекта на 3 части. Client, Server и Protocol.
Часть "Protocol" используют и client, и server, так что это общая часть проекта. 

Proto-файлов может быть несколько, в каждом может быть описан свой сервис и сообщения используемые этим сервисом. 

```protobuf
// пример proto-файла; пусть это будет howto.proto

package howto; // имя namespace для C++

/// cc_generic_services говорит протобуферу, 
/// что нужно сгенерировать сервис; 
/// без этого параметра будет сгенерирован исходник, 
/// описывающий только сообщения, но не service
option cc_generic_services = true; 

message request_message { 
    optional string name = 1;
}

message response_message { 
    optional string hello = 1;
}

service hello_service {
    rpc send_hello( request_message ) returns ( response_message );
}

```

>
```
Далее по тексту: 
    "сообщение" — сообщение описанное в proto-файле как message
    "сервис" – класс, реализующий методы (rpc) сервиса (service), описанные в proto-файле. 
```
>

в файле ```howto.proto``` описано 2 сообщения: ```request_message``` и ```response_message```. Первое — сообщение-запрос, которое нужно будет *заполнить клиенту перед* тем, как отправить на сторону сервера. Второе – ответ и его *заполнит уже сервер*, когда выполнит вызов клиента. Это сообщение получит клиент.

После генерации мы получим 2 файла  howto.pb.h и howto.pb.cc (если наш файл называется howto.proto), которые должны будут включены в проект. Я использую cmake для генерирования и автоматического включения этих файлов в состав проекта, так мне не нужно думать о перегенерации файлов при изменении *.proto. 

Если в протофайле задана опция  cc_generic_services (её можно так же задать в командной строке protoc), то помимо сообщений будут сгенерированы классы для работы с сервисами. По два на каждый.

**Первый класс** (назовем его интерфейсом для сервиса) — класс для реализации обработчика запросов. Этот класс носит ровно то имя, которое было задано в proto-файле. То есть в нашем случае это будет класс  hello_service (howto ::hello_service, если с именем namespace). Этот класс содержит **виртуальные** методы, которые реализуют методы rpc, описанные сервисе. У нас будет один метод –  send_hello. В С++ он будет выглядеть так (howto.pb.h):

```cpp
class hello_service : public ::google::protobuf::Service {
.........
    virtual void send_hello(::google::protobuf::RpcController* controller,
                       const ::howto::request_message* request,
                       ::howto::response_message* response,
                       ::google::protobuf::Closure* done);
.........
};

```

Помимо сообщений запроса и ответа, описанных в протофайле protoc добавил еще 2 сущности **controller** и **done**(замыкание), которые являются служебными и используются на стороне как клиента, так и сервера. О них ниже.

Нагенерённые методы не являются чисто-виртуальными, у каждого из них есть реализация в  соответвующем *.pb.cc файле. Реализация не пустая, в ней устанавливается ошибка в controller. Ошибка говорит о том, что данный метод  (send_hello), данного сервиса (hello_service) не реализован. Такая заглушка. Далее эта ошибка передается клиенту. 

```cpp
void hello_service::send_hello(::google::protobuf::RpcController* controller,
                       const ::howto::request_message* request,
                       ::howto::response_message* response,
                       ::google::protobuf::Closure* done) {
  controller->SetFailed("Method send_hello() not implemented.");
  done->Run();
}
```

Этот класс мы будем использовать на стороне, которая хочет реализовать этот сервис у себя. Это не обязательно должна быть серверная сторона. Это может и сервер, и клиент, и обе стороны одновременно. Использование заключается в наследовании от данного класса и переопределении виртуальных методов. Что нужно еще см. раздел **Server**.

**Второй класс**, который получается из описанного в *.proto – это Stub-класс и он будет использоваться для доступа к сервису на другой стороне, которая этот сервис реализует. В нашем случае этот класс будет называться hello_service_Stub. Этот класс уже не является интерфейсным, то есть у него нет виртуальных методов. В конструкторе этот класс принимает указатель на RpcChannel, котороый использует для отправки запросов. О  RpcChannel, опять же, ниже. Методы имеют ровно тот же вид, что и в классе-интерфейсе. То есть 


```cpp
class hello_service_Stub : public ::google::protobuf::Service {
.........
    void send_hello(::google::protobuf::RpcController* controller,
                    const ::howto::request_message* request,
                    ::howto::response_message* response,
                    ::google::protobuf::Closure* done);
.........
};

```
Но уже с готовой реализацией.

####Итого: 

Сторона, которая хочет реализовать некий метод rpc, описанный в *.proto файлах, создает класс-наследник от класса-интерфейса и реализовывает нужные методы. 

Сторона, которая хочет получить доступ к методам сервиса первой стороны, использует Stub-класс.

Сторона реализующая сервис (не обязательно сервер!):
```cpp 

/// наследуемся от howto::hello_service
class  hello_service_impl: public howto::hello_service { 
    void send_hello(::google::protobuf::RpcController* controller,
                    const ::howto::request_message* request,
                    ::howto::response_message* response,
                    ::google::protobuf::Closure* done) override
    { 
        /// вход в обработчик запроса

        std::ostringstream oss;

        /// возьмем строку из запроса
        oss << "Hello " << request->hello( )  
            << " from hello_service_impl::send_hello!";
        
        /// поместим результат в ответ
        response->set_hello( oss.str( ) );
        
        /// done->Run( ) отошлет ответ!
        done->Run( ); 
    }
};

```

Сторона-клиент
```cpp
howto::hello_service_Stub stub(channel); /// пользуем Stub-класс
howto::request_message  req;             /// сообщение-запрос
howto::response_message res;             /// сообщение-результат
req.set_name( "%USERNAME%" );  /// установим значение поля name в запросе

/// тут параметры controller и done, (как и запрос, ответ) 
/// могут быть NULL
stub.send_hello( NULL, &req, &res, NULL ); /// вызов удаленного метода 

/// теперь в res.hello( ) у нас то, что написала туда другая сторона.
std::cout <<  res.hello( ) << std::endl; 
```

результат клиента
```console
Hello %USERNAME% from hello_service_impl::send_hello!
```

Клиент и сервер могут находиться как на одном хосте, так и на разных континентах.


####Опции сервисов и методов.

TODO: написать

##Common

> Та самая часть библиотеки, которая используется и клиентом, и сервером. Тут я опишу некоторые общие сущности, которые используются в библиотеке.

####Управление потоками и диспетчеры.

Для работы сервера и клиента требуется наличия boost::asio::io_service, который используется для асинхронных событий ввода-вывода, а так же для исполнения удаленных вызовов. 

В библиотеке имеется 2 класса для запуска потоков, работающих с io_service: 
vtrc::common::thread_pool — пул потоков с одним io_service внутри.
vtrc::common::pool_pair — пул с парой  io_service. Реализован на базе  thread_pool.

**thread_pool** – запускает заданное количество потоков и обрабатывает в них задания из io_service. Потоки можно добавлять и прерывать. 

thread_pool::get_io_service вернет ссылку на io_service

**pool_pair** – сложнее. Данный класс может создавать как один, так и два io_service. Классы, которые могут принимать в конструкторе ссылку на данный класс используют ```pool_pair::get_io_service``` для IO операций и ```pool_pair::get_rpc_service``` для вызова методов сервисов. vtrc::server::application, например, в одном из своих конструкторов может принимать как ссылку на один единственней io_service, так и на  pool_pair.

##Server

Первое, что нужно для реализации серверной части — это класс-наследник от vtrc::server::application. 

Application — это основной механизм общения библиотеки с приложением-сервром. Application хранит в себе список всех соединений. При помощи application библиотека получает экземпляры реализаций сервисов, настраивает соединения. 

####Конструкторы

У  application имеется 4 (четыре) конструктора.

````cpp
        application( );
````

Говорит классу, что он должен сам создать себе io_service. Один. Этот io_service  будет использован как для IO операций, так и для вызовов. В этом случае пользователь библиотеки сам определяет как запускать обработку данного io_service. ```boost::asio::io_service &get_io_service( );``` и ```boost::asio::io_service &get_rpc_service( );``` возвращают ссылку на один и тот же объект.


````cpp
        application( boost::asio::io_service &ios );
        application( boost::asio::io_service &ios,
                     boost::asio::io_service &rpc_ios);

````

Принимают ссылки на  io_service. 

В первом случае это будет один единственный io_service, и работа с ним будет аналогична тому, как описано выше. Подразумевается, что у пользователя уже есть io_service, который будет использован сервером. ```boost::asio::io_service &get_io_service( );``` и ```boost::asio::io_service &get_rpc_service( );``` будут возвращать ссылки на соответствующий io_service;

В этих 2 случаях, пользователь сам определеят как и когда запускать обработку для этих io_service.

Вариант с 2 параметрами (2 ссылки на io_service; ios, rpc_ios ), говорит приложению использовать раздельную схему обработки IO и вызовов. Это позволяет более тонко контролировать ресурсы системы. Так же это позволяет избежать "забивания" одного диспетчера и невозможности обрабатывать ввод-вывод.

````cpp
        application( common::pool_pair &pools );

````

Практически аналогичен версси с 2 ссылками на io_service. Из pools приложение получает io_service для IO и для rpc. Это может быть один и тот же io_service.

Обычно, common::pool_pair сам запускает обработку  io_service (метод run/run_one).

```cpp
/// пара io_service и потоки для них
vtrc::common::pool_pair pp(2, 4);

/// application_impl — наследник от vtrc::server::application
application_impl app(pp);

```

Создаст 2 потока для обработки IO и 4 для вызовов rpc.


```cpp
/// один io_service 8 потоков обработки
vtrc::common::pool_pair pp( 8 );

/// application_impl — наследник от vtrc::server::application
/// будет работать на одном io_service
application_impl app(pp);

```

В этом случае будет 8 потоков, обрабатывающих один io_service.


####Дальше

У vtrc::server::application есть 3 виртуальных метода.

```cpp

virtual std::string get_session_key( common::connection_iface* conn,
                                     const std::string &id);

virtual void configure_session( common::connection_iface* connection,
                                vtrc_rpc::session_options &opts );

virtual vtrc::shared_ptr<common::rpc_service_wrapper>
                 get_service_by_name( common::connection_iface* connection,
                                      const std::string &service_name );
```

**get_session_key** - вызывается библиотекой при использовании клиентом ключа для соединения. Библиотека поддерживает совсем простую базовую авторизацию клиентов. **get_session_key** может вернуть ключ для данного соединения (1ый параметр). Это может быть общий ключ, либо уникальный для каждого клиента. На стороне клиента этот ключ так же известен. При несовпадении ключей, коммуникация будет невозможна. Второй параметр — это id клиента. Это может быть имя, номер, что угодно. 

**configure_session** - вызывается после успешного соединения клиента. В качестве параметров передается интерфейс соединения (1ый параметр) и объект опций (2ой параметр). vtrc_rpc::session_options — сообщение описанное в vtrc-rpc-lowlevel.proto.

```proto
message session_options {
    optional uint32 max_active_calls    = 1 [default = 5];
    optional uint32 max_message_length  = 2 [default = 65536];
    optional uint32 max_total_calls     = 3 [default = 20];
    optional uint32 max_stack_size      = 4 [default = 64];
    optional uint32 read_buffer_size    = 5 [default = 4096];
}
```
Эти опции можно поменять для конкретного соединения. После установки опций, клиент так же получит сведения о них. 

**get_service_by_name** - основной метод для библиотеки. Благодаря этому методу серверная часть получает экземпляр класса, описанного сервиса. В качестве параметров передается интерфейс соединения, а так же имя нужного сервиса. В случае успеха вызов возвращает умный указатель на **common::rpc_service_wrapper** – обёртку над google::protobuf::Service. Если запрашиваемый сервис недоступен для соединения, то  **get_service_by_name** может вернуть ```vtrc::shared_ptr<common::rpc_service_wrapper>( )```(пустой указатель), либо кинуть исключение. В первом случае клиент получит ошибку о недоступности данного сервиса, во втором случае клиент получит INTERNAL_ERROR с текстом из std::exception::what( ); 
**get_service_by_name**, в случае успеха, вызывается только один раз, далее указатель сохраняется и используется для последующих вызовов. То есть инициализация сервиса для соединения на серверной стороне является ленивой. Полученные объекты "живут" до тех пора, пока "живёт" соединение.

> стоит отметить, что библиотека обрабатывает исключения-наследники от std::exception и что-то другое будет проглочено и отправлено на другую сторону с текстом '...'.

####Возвращаясь к hello_service_impl.

Далее пример application для серверной части, реализующей сервис ```hello_service```, описанный в резделе **Protocol**. 


```cpp

/// наследуемся от howto::hello_service
class  hello_service_impl: public howto::hello_service { 
    void send_hello(::google::protobuf::RpcController* controller,
                    const ::howto::request_message* request,
                    ::howto::response_message* response,
                    ::google::protobuf::Closure* done) override
    { 
        /// вход в обработчик запроса

        std::ostringstream oss;

        /// возьмем строку из запроса
        oss << "Hello " << request->hello( )  
            << " from hello_service_impl::send_hello!";

        /// поместим результат в ответ
        response->set_hello( oss.str( ) );

        /// done->Run( ) отошлет ответ!
        done->Run( ); 
    }
};
............

class hello_application: vtrc::server::application {
public:
    hello_application( vtrc::common::pool_pair &pp )
        :vtrc::server::application(pp)
    { }

    vtrc::shared_ptr<common::rpc_service_wrapper>
                 get_service_by_name( common::connection_iface* connection,
                                      const std::string &service_name )
    {
        /// проверим, соответсвует ли имя запрашиваемого сервиса, нашему hello_serve
        if( service_name == hello_service_impl::descriptor( )->full_name( ) ) {

             /// создаем экземпляр
             hello_service_impl *new_impl = new  hello_service_impl;

             /// создаем и возвращем обертку
             return vtrc::make_shared<common::rpc_service_wrapper>( new_impl );
        }
        /// вернем пустой указатель. Клиент получит ошибку о недоступности сервиса
        return vtrc::shared_ptr<common::rpc_service_wrapper>( );
    }
    
}
```
еще похожий пример использования этого метода можно увидеть в примерах: https://github.com/newenclave/vtrc/blob/master/examples/lukki-db/server/lukki-db-application.cpp#L393

Возможно в будущем метод get_service_by_name  будет заменен на что-то более удобное.


Обертка common::rpc_service_wrapper сделана для бóльшей гибкости. При помощи нее можно, например, отказывать клиенту в некоторых методах сервиса. 

```cpp

class my_rpc_wrapper: public common::rpc_service_wrapper
{
    common::connection_iface* connection_;
public:
    my_rpc_wrapper( google::protobuf::Service *my_serv, common::connection_iface* c )
        :common::rpc_service_wrapper(my_serv)
        ,connection_(c)
    { }

    const google::protobuf::MethodDescriptor *get_method( const std::string &name ) const
    {
        /// find_method - protected метод; 
        /// по имени найдет описатель (Descriptor) метода
        const google::protobuf::MethodDescriptor *result = find_method( name );

        /// какой-то внешний вызов, 
        /// который определит, что этому соединению можно исполнить метод 'name'
        if( result && !::is_valid_method_for_connection( result, connection_ ) ) {
            /// сбросим. Сделаем вид, что нет такого метода.
            result = NULL; /// клиенту уйдет ошибка о том, что метод недоступен

            /// Еще правильный и доступный вариант — исключение
            /// throw std::logic_error( "Access denied!" );
            /// Клиент получит ошибку с текстом what( );
        } 
        return result;
    }

};

```

#####Итого: 

Реализацию сервиса библиотека получает из метода **get_service_by_name** от наследника vtrc::server::application. Реализация завернута в обертку common::rpc_service_wrapper (либо его наследника). 
Метод сервиса получается уже из обертки. Методом **get_method** этой обертки. Защищенный метод **find_method** находит по имени описатель метода сервиса.


Теперь у нас практически все готово к тому, чтобы серверу начать свою работу. Итак ... 

####Слушатель (listener)

Слушатель это сущность, которая открывает точку на стороне сервера и принимает соединения. Можно считать, что слушатель — это порт для доступа к сервисам. 

##Client



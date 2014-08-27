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

Для описания протокола используются proto-файлы, из которых после, посредством Potobuf, генерируются исходники. (Если читатель не знаком с Protocol Buffers, то самое время ознакомиться тут https://code.google.com/p/protobuf/; Обычно хватает часа, а может и меньше, чтоб понять что это, зачем и как этим что-то описать и получить).

В примерах (https://github.com/newenclave/vtrc/tree/master/examples) я использую разделение проекта на 3 части. Client, Server и Protocol.
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
        oss << "Hello " << request->name( )  
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

####Исключения. 

Все исключительные ситуации в библиотеке сигнализируются при помощи исключений. Все внутренние исключения - наследники от std::exception.

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

**get_session_key** - вызывается библиотекой при использовании клиентом ключа для соединения. Библиотека поддерживает совсем простую базовую авторизацию клиентов. **get_session_key** может вернуть ключ для данного соединения (1ый параметр). Это может быть общий ключ, либо уникальный для каждого клиента. На стороне клиента этот ключ так же известен. При несовпадении ключей, коммуникация будет невозможна. Второй параметр — это id клиента. Это может быть имя, номер, что угодно. ID устанавливается методом ```set_session_id``` или методом ```set_session_key``` клиента. (см. раздел "Client").

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
        oss << "Hello " << request->name( )  
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

Слушатель это сущность, которая открывает точку на стороне сервера и принимает соединения. Интерфейс слушателя описан в серерной части библиотеки в vtrc-listener.h 

listener имеет очень простой интерфейс. Основные методы: **start**, **stop** и **name**. Названия методов говорят сами за себя. 

**start** - запускает слушателя

**stop** - останавливает

**name** - возвращает имя. Имя зависит от типа слушателя. (например ```tcp://:::55555``` для tcp6 точки с портом 55555).


В заголовке **vtrc-server/vtrc-listener-tcp.h** описано 2 функции-фабрики, которые создают указатель на экземпляр слушателя для TCP. 

```cpp
    namespace listeners { namespace tcp {

        listener_sptr create( application &app,
                              const std::string &address,
                              unsigned short service,
                              bool tcp_nodelay = false );

        listener_sptr create( application &app,
                              const vtrc_rpc::session_options &opts,
                              const std::string &address,
                              unsigned short service,
                              bool tcp_nodelay = false );
    }}
```

Первый принимаемый параметр — ссылка на экземпляр application (описан выше). Из него слушатель получает нужный io_service, а так же конфигурирует соединение.

Первый вариант создает точку с настройками для точки по-умолчанию, второму варианту эти настройки можно передать явно. 

Параметр *address* содержит локальный IP адрес для данной точки. Например: ```127.0.0.1```, ```0.0.0.0```. Или IPv6 ```::1```, ```::``` и т.д.

*service* — это порт, который следует открыть данной точке.

Параметр  tcp_nodelay отключает действие *алгоритма Нейгла* для всех соединений данной точки. Влияние этого алгоритма можно увидеть в примере **stress**, входящий в состав библиотеки. Думаю не сильно ошибусь, если скажу, что в 95% случаев этот параметр может оставаться в *false*.

```cpp
    vtrc::shared_ptr<server::listener> local_net_tcp =
         server::listeners::tcp::create( app, "192.168.1.100", 32344);

```

Откроет для приема 32344 порт на интерфейсе с адресом  192.168.1.100 (это может быть, например, локальная сеть).

Заголовок **vtrc-server/vtrc-listener-local.h** предоставляет функции для создания локальных слушателей. Для систем Windows это точки, которые открывают Named pipe для клиентов. В системах, поддерживающих POSIX — это Unix domain socket (POSIX Local IPC Sockets).

```cpp
    namespace listeners { namespace local {
        listener_sptr create( application &app, const std::string &name );
        listener_sptr create( application &app,
                              const vtrc_rpc::session_options &opts,
                              const std::string &name );
    }}
```

Парарметр *name* содержить путь к POSIX сокету, либо имя пайпа (для windows).


```cpp
    vtrc::shared_ptr<server::listener> local_socket =
         server::listeners::local::create( app, "/home/sandbox/local.sock");

```

или для windows

```cpp
    vtrc::shared_ptr<server::listener> local_socket =
         server::listeners::local::create( app, "\\\\.\\pipe\\localpipe");

```

Теперь запрос на соединение будет получен, обработан.

Кроме запуска и остановки слушатель имеет возможность сообщить внешнему миру о некоторых своих событиях. 

Для этого используются сигналы из библиотеки boost (boost::signal2). Пока таких сигналов 5:


```cpp
    VTRC_DECLARE_SIGNAL( on_start, void ( ) );
```
Генерируется при успешном старте слушателя. 

```cpp
    VTRC_DECLARE_SIGNAL( on_stop,  void ( ) );
```
Генерируется при остановке слушателя. 


```cpp
    VTRC_DECLARE_SIGNAL( on_accept_failed, void ( const boost::system::error_code &err ) );
```
Ошибка приема нового клиента. В ```err``` передается значение ошибки. В обработчике сигнала можно остановить слушатель, либо игнорировать. По ситуации.

```cpp
    VTRC_DECLARE_SIGNAL( on_new_connection, void ( const common::connection_iface * ) );
```
Новое соединение. Сигнал генерируется только после успешного соединения клиента, умпешного обмена ключами, если таковые заданы. 

```cpp
    VTRC_DECLARE_SIGNAL( on_stop_connection, void ( const common::connection_iface * ) );

```
Закрытие соединения. Вызывается ПОСЛЕ закрытия соединения. Объект класса connection_iface еще живой, но уже не имеет соединения. 

Все сигналы от одного слушателя вызываются неконкурентно. То есть 2 сигнала не могут быть вызваны одновременно из 2 разных потоков. Однако 2 разных слушателя могут сгенерировать даже один и тот же сигнал из разных потоков одновременно.


###Пример использования сигналов:

Примеры можно увидеть, например, тут: https://github.com/newenclave/vtrc/blob/master/examples/calculator/server/main.cpp#L185

В примере, при успешном создании слушателя, application подписывается на его сигналы ```on_new_connection``` и ```on_stop_connection```. 


##Client

С клиентской стороны все немного проще. Клиенту нужен только канал, который будет соеденен с удаленной сторой. 

Как и серверные слушатели, клиент требует наличия io_service для своей работы. 

```cpp
    common::pool_pair pp( 1, 1 );

    client::vtrc_client_sptr client = client::vtrc_client::create( pp );
```

```create``` является статическим методом ```client::vtrc_client``` и сделан для того, чтоб создавать правильный указатель на клиента и инициализировать его (клиента). Создавать клиента на стеке, либо вызовом ```new client::vtrc_client(pp);``` нельзя

Как и в случае слушателей серверной стороны, у клиента есть свои сигналы. Всего таких сигналов пока 4 

```cpp
    VTRC_DECLARE_SIGNAL( on_connect, void( ) );
```

Возникает при соединении клиента с удаленной стороной. После этого сигнала клиент соединен с сервером, но не готов еще работать. 

```cpp
    VTRC_DECLARE_SIGNAL( on_init_error, 
                         void( const vtrc_errors::container &, 
                         const char * ) );
```
Ошибка инициализации. Сигналит когда клиент уже соединен, но не смог договориться с сервером. Например подсовывает ему некорректный ключ. Первый параметр ```const vtrc_errors::container &``` содержит в себе сведения об ошибке, второй ```const char *``` дополнительное сообщение.

```cpp
    VTRC_DECLARE_SIGNAL( on_ready, void( ) );
```
Готов! После этого сигнала клиент готов к работе с удаленной стороной. Можно создавать каналы, ожидать события и использовать вызовы.

```cpp
    VTRC_DECLARE_SIGNAL( on_disconnect, void( ) );
```
Клиент/сервер разорвали соединение.


Пример: https://github.com/newenclave/vtrc/blob/master/examples/lukki-db/client/client.cpp#L178

```cpp

/// некоторые обработчики некоторых сигналов.
namespace {
    void on_connect( )
    {
        std::cout << "connect...";
    }
    void on_ready( )
    {
        std::cout << "ready...";
    }
    void on_disconnect( )
    {
        std::cout << "disconnect...";
    }
}
```

подпишемся на сигналы клиента. Это следует сделать **ДО попытки соединиться с удаленной стороной** иначе сигналы можно “пропустить”.

```cpp
    client->on_connect_connect ( on_connect );
    client->on_ready_connect ( on_ready );
    client->on_disconnect_connect( on_disconnect );
```

Теперь клиент готов пойти и попросить у сервера немного сервисов.

Допустим по адресу 192.168.3.1:55550 есть некий сервер. Соединимся с ним

```cpp
    client->connect( "192.168.3.1", 55550 );
```

В случае успеха мы получим сигналы (по порядку) ```on_connect```, ```on_ready```. В случае недоступности хоста - получим exception.

Кроме вызова, который соединяется с сервером TCP

```cpp
    void connect( const std::string &address,
                  unsigned short service,
                  bool tcp_nodelay = false );
```

У клиента есть вызов, который соединяется с локальной точкой
```cpp
    void connect( const std::string &local_name );
```
Кроме того, есть асинхронные варианты вызовов:

```cpp
    void async_connect( const std::string &local_name,
                        common::system_closure_type closure);

    void async_connect( const std::string &address,
                        unsigned short     service,
                        common::system_closure_type closure,
                        bool tcp_nodelay = false );
```

Которые принимают, помимо обычных параметров, еще и функцию-замыкание, которая будет дернута по завершению соединения, а сам вызов вернёт управление сразу.

Теперь, когда мы успешно соединены, можно использовать то, что нам предоставил протобуфер, когда сгенерил исходники из *.proto файлов. 
Как сообщалось ранее, для описаных сервисов генерируется 2 класса. Второй класс, который называется Stub нам и нужен для исполнения удаленного вызова. Stub-классы в качестве параметра в конструктор требуют вот такую вот сущность: 
```::google::protobuf::RpcChannel *channel)```.

Например, сгенерированный код Stub-класса из примера lukki-db https://github.com/newenclave/vtrc/tree/master/examples/lukki-db

```cpp
class lukki_db_Stub : public lukki_db {
 public:
  lukki_db_Stub(::google::protobuf::RpcChannel* channel);
  lukki_db_Stub(::google::protobuf::RpcChannel* channel,
                ::google::protobuf::Service::ChannelOwnership ownership);
....
}
```
Второй конструктор может сказать Stub-классу, о том, что он сам владеет каналом и должен его сам удалить.

Этот RpcChannel есть абстракция. И наследника от этого класса (реализацию канала) может вернуть клиент: 

Два вызова 

```cpp
    rpc_channel_c *create_channel( );
    rpc_channel_c *create_channel( unsigned flags );
```
Вызовы создают новый канал.
    
    Тут "создают канал" не значит, что клиент будет куда-то опять соединяться.

    !!! Вызовы возвращают RAW-указатель и тот, кто создал канал сам должен его удалить.

Второй вариант принимает некоторые флаги, которые будет отвечать за работу канала. Флаги описаны в части "common" библиотеки в классе "rpc_channel" https://github.com/newenclave/vtrc/blob/master/vtrc-common/vtrc-rpc-channel.h#L26 
Значения флагов: 

```cpp
 DEFAULT = 0
```
канал по-умолчанию. Вызовы будут регулироваться опциями, описанными в прото-файлах

```cpp
,DISABLE_WAIT = 1
```
Создаст канал с отключенным ожиданием, даже если вызов рассчитан на то, чтоб вернуть какие-то данные, возвращать он их не будет. Любой вызов, сделанный поверх этого канала вернут управление сразу.

```cpp
,USE_CONTEXT_CALL = 1 << 1
```
Эта опция говорит о том, что любой вызов сделанный поверх такого канала будет пытаться найти контекст текущего вызова и отработать на другой стороне в контексте этого вызова. Эта опция создает канал, по которому можно будет делать CALLBACK'и.

####И снова hello сервис. И пример использования на стороне клиента.

```cpp
    /// Тут наш клиент уже успешно подключен.
    /// создадим канал, и обернем его в  std::unique_ptr
    std::unique_ptr<vtrc::common::rpc_channel> channel(client->create_channel( ));

    /// создадим  Stub-класс для вызовов.
    hello_service_Stub hello(channel.get( ));
    
    /// создадим сообщения для запроса и для результата
    howto::request_message  req;
    howto::response_message res;
    
    /// Сервер ожидает, что в запросе будет установлено имя, 
    /// которое он будет использовать для ответа. Установим
    req.set_name( "%USERNAME%" );

    /// сделаем вызов!
    hello.send_hello( NULL, &req, &res, NULL );

    /// если все прошло удачно, в  res мы будем иметь ответ.
    std::cout <<  res.hello( ) << std::endl;

```

    В случае vtrc, параметры для вызовов (любой из четырех) не являются обязательными. Могут быть NULL.
    Если нет параметра-запроса, то будет отправлен вызов с пустым запросом
    !!! Если нет параметра-ответа, то другая сторона НЕ БУДЕТ даже сериализовать сообщение с ответом. 
    Кроме того на другой стороне есть возможность узнать, что ответ не ожидается.
    Это может помочь избежать ненужных вычислений, например.

Для того, чтоб не писать каждый раз такую длинную лапшу для каждого вызова. Библиотека имеет обертку: https://github.com/newenclave/vtrc/blob/master/vtrc-common/vtrc-stub-wrapper.h

Пример использования обертки можно увидеть в любом из примеров. Например тут: https://github.com/newenclave/vtrc/blob/master/examples/lukki-db/client/lukki-db-impl.cpp 

Для нашего "Hello word!"'a это будет выглядеть примерно так: 

```cpp
        /// Тут наш клиент уже успешно подключен.
    /// создадим канал, и обернем его в  std::unique_ptr
    std::unique_ptr<vtrc::common::rpc_channel> channel(client->create_channel( ));

    /// создадим обертку Stub-класса для вызовов.
    vtrc::common::stub_wrapper<hello_service_Stub> hello(channel.get( ));
    
    /// создадим сообщения для запроса и для результата
    howto::request_message  req;
    howto::response_message res;
    
    /// Сервер ожидает, что в запросе будет установлено имя, 
    /// которое он будет использовать для ответа. Установим
    req.set_name( "%USERNAME%" );

    /// сделаем вызов!
    hello.call( &hello_service_Stub::send_hello, &req, &res );

    /// если все прошло удачно, в  response мы будем иметь ответ.
    std::cout <<  res.hello( ) << std::endl;

```

BTW обертка умеет хранить в себе и ```vtrc::shared_ptr```

Еще замечание: Клиент в любой момент времени можно отключить ```disconnect( );``` и подключить снова. Либо просто переподключить методом ```connect/async_connect```. В любом случае разрыва соединения каналы, созданные клиентом станут инвалидными и любой вызов, сделанный при помощи такого канала выкинет исключение.

###ID клиента, ключ.

Вызовы 
```cpp
    void set_session_key( const std::string &id, const std::string &key );
    void set_session_key( const std::string &key );
    void set_session_id ( const std::string &id );
```

Устанавливают id и ключ для соединения с сервером. Можно установить только ключ или только идентификатор. На стороне сервера ID клиента станет известен во время инициализации соединения, а так же его можно будет в любой момент получить методом класса ```common::connection_iface::id( )```. Ключ используется для настройки шифрования и если он не совпадет с тем, что выдаст клиенту сервер (см. описание ```server::application::get_session_key``` выше), соединение будет невозможною. Сервер такое соединение скинет, а клиент получит сигнал ```on_init_error```.



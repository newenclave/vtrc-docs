/* UTF8 */

## HELLO


Простой пример использования библиотечки.


##Protocol 

Протокол декларирован в proto-файле (https://github.com/newenclave/vtrc/blob/master/examples/hello/protocol/hello.proto). 

```protobuf

package howto; // namespace для С++
option cc_generic_services = true; // говорим протобуферу сгенерировать 
                                   // service hello_service 

message request_message {          // сообщение - запрос
    optional string name = 1;
}

message response_message {         // сообщение - ответ
    optional string hello = 1;
}

service hello_service {            // описание сервиса. 
    // в сервисе всего один вызов, который принимает сообщение request_message
    // и результат помещает в сообщение response_message
    rpc send_hello( request_message ) returns ( response_message );
}

```


##Server 

Файлы *.h
```cpp
#include "vtrc-server/vtrc-application.h"  // для основного класса server::application
#include "vtrc-server/vtrc-listener-tcp.h" // слушатель TCP

#include "vtrc-common/vtrc-connection-iface.h" // информация о соединении
#include "vtrc-common/vtrc-closure-holder.h"   // RAII класс для замыкания в вызове 
#include "vtrc-common/vtrc-thread-pool.h"      // управление потоками и сервисом io_service

#include "protocol/hello.pb.h"          // hello protocol. Сгенерированный файл из  hello.proto
#include "google/protobuf/descriptor.h" // для descriptor( )->full_name( ) 
#include "boost/lexical_cast.hpp"       // для приведения номера порта из командной строки. 


using namespace vtrc; /// чтоб не писать лишний раз

```

Первое, что описано в файле исходнике сервера - это класс-сервис, наследник от сгенерированного howto::hello_service

```cpp
class  hello_service_impl: public howto::hello_service {
    
    /// будем сохранять информацию о клиенте, владеющим этим классом
    common::connection_iface *cl_;

    /// обработчик вызова, описанного в ptoto-файле как 
    ///     rpc send_hello( request_message ) returns ( response_message );
    void send_hello(::google::protobuf::RpcController*  /*controller*/, // не нужен в данном примере
            const ::howto::request_message*     request,  // запрос
            ::howto::response_message*          response, // сообщение с ответом
            ::google::protobuf::Closure*        done) /*override*/  // сигнал об исполнении
    {
        common::closure_holder ch( done ); /// RAII. в деструкторе будет вызван done->Run( );
        
        std::ostringstream oss;

        // создадим строчку с приветствием, в которую добавим информацию о клиете,
        // который пользуется нашим сервисом 
        // cl_->name( ) вернет имя транспорнтной точки, которой пользуется клиент
        oss << "Hello " << request->name( )
            << " from hello_service_impl::send_hello!\n"
            << "Your transport name is '"
            << cl_->name( ) << "'.\nHave a nice day.";
        
        /// установим сообщению-ответу строку с результатом
        response->set_hello( oss.str( ) );
    }

public:
    
    // constructor
    hello_service_impl( common::connection_iface *cl )
        :cl_(cl)
    { }
    
    /// просто удобная обертка для получения имени данного сервиса.
    static std::string const &service_name(  )
    {
        // вернет то, что сгенерировал protobuf
        return howto::hello_service::descriptor( )->full_name( );
    }

};
```

#### Application

Второй класс в исходнике - основное приложение. Наследник от vtrc::server::application
```cpp
class hello_application: public server::application {
    
    /// удобный typedef.
    typedef common::rpc_service_wrapper     wrapper_type;
    typedef vtrc::shared_ptr<wrapper_type>  wrapper_sptr;

public:

    /// принимает класс common::thread_pool и берет из него ссылку на io_service
    ///         который необходим для работы приложения.
    hello_application( common::thread_pool &tp )
        :server::application(tp.get_io_service( ))
    { }
    
    /// Основной метод!
    /// принимает интерфейс клиента и имя запрашиваемого сервиса.
    /// возвращает обертку, содержащую экземпляр нашего сервиса (hello_service_impl)
    wrapper_sptr get_service_by_name( common::connection_iface* connection,
                                      const std::string &service_name )
    {
        /// проверим действительно ли клиент запросил наш сервис
        /// и если да, то создадим экземпляр hello_service_impl и обертку для него
        if( service_name == hello_service_impl::service_name( ) ) {
             
             /// сервис 
             hello_service_impl *new_impl = new hello_service_impl(connection);
            
             /// вернем обертку.
             return vtrc::make_shared<wrapper_type>( new_impl );

        }
        /// клиент запросил что-то не то. вернем пустой указатель, 
        /// который обозначит отсутствие сервиса и отправит клиенту ошибку
        return wrapper_sptr( );
    }

};
```

Теперь у нас есть все, что нужно для исполнения удаленного вызова для клиентов.
Осталось только добавить возможность пользоваться этим всем.

#### main

Функция main создает наш hello_application и запускает слушателя, который будет принимать клиентов и передавать их на обслуживание приложению (server::application)

```cpp
int main( int argc, const char **argv )
{

    /// значение по-умолчанию
    const char *address = "127.0.0.1";
    unsigned short port = 56560;

    /// если что-то передали в командную строку, попробуем это применить в качестве:
    if( argc > 2 ) {
        /// 2 параметра IP PORT
        address = argv[1];
        port = boost::lexical_cast<unsigned short>( argv[2] );
    } else if( argc > 1 ) {
        /// 1 параметр PORT
        port = boost::lexical_cast<unsigned short>( argv[1] );
    }

    /// экземпляр класса для управления потоками и io_service
    /// по-умолчанию он не запускает никаких потоков, а просто создает io_service
    /// который доступен через метод get_io_service( )
    /// можно добавить сразу несколько потоков, передав их количество в конструктор
    /// Например common::thread_pool tp( 2 );
    common::thread_pool tp;
    
    /// экземпляр нашего приложения
    hello_application app( tp );

    /// в случае некорректных данных (неправильная строка с IP)
    /// либо в случае системных ошибок (например невозможность открыть порт, потому что уже занят)
    /// будет сгенерировано исключение
    try {
        
        /// создаем слушателя, передаем ему ссылку на наше приложение
        /// create возвращает shared_ptr<server::listener> !!!
        /// то есть сделать указатель уникальным нельзя
        vtrc::shared_ptr<server::listener>
                tcp( server::listeners::tcp::create( app, address, port ) );
        
        /// start! тут будет открыт порт и начнут приниматься клиенты.
        tcp->start( );

        /// запустим цикл обработки событий в текущем потоке
        /// таким образом наш сервер будет крутиться всего в одном потоке - 
        /// потоке функции main.
        /// НО! 
        /// В системах Windows будет второй поток, который управляет таймерами
        tp.attach( );

        /// Кстати.
        /// При выходе из области видиммости слушатель будет остановлен и уничтожен
        /// поэтому выносить tp.attach( ) за try { } catch не имеет смысла. 
        /// сервер просто не будет работать

    } catch( const std::exception &ex ) {
        /// что-то неполучилось. Скажем в консоль об этом
        std::cerr << "Hello, world failed: " << ex.what( ) << "\n";
    }
    
    /// дождемся завершения всех потоков, которые были созданы через common::thread_pool
    /// в нашем случае это лишнее, потому как у нас дополнительных потоков нет
    /// оставил в качестве примера
    tp.join_all( );

    /// make valgrind happy.
    google::protobuf::ShutdownProtobufLibrary( );

    return 0;
}
```
Теперь можно запустить сервер

    ./hello_server               # запустить сервер на 127.0.0.1:56560
    ./hello_server 12345         # запустить сервер на 127.0.0.1:12345
    ./hello_server 0.0.0.0 12345 # запустить сервер на 0.0.0.0:12345

С сервером примера всё.

##Client

С клиентом всё немного проще. Клиент должен только соединиться и предоставить канал для Stub-классов сервисов. 

####Заголовки: 

```cpp

#include "vtrc-client/vtrc-client.h"       /// client::vtrc_client сам клиент
#include "vtrc-common/vtrc-thread-pool.h"  /// Управлятор потоками 
#include "vtrc-common/vtrc-stub-wrapper.h" /// Обертка для использоания Stub-классов
                                           ///  без обетки можно вполне обойтись

#include "protocol/hello.pb.h"  /// сгенерированные классы для работы с протоколом
 
#include "boost/lexical_cast.hpp" /// для приведения порта из строки в число

using namespace vtrc; /// чтоб не писать постоянно

```
Далее объявлены функции, которые есть реакция на события клиента.

```cpp
namespace {

    /// соединён
    void on_connect( )
    {
        std::cout << "connect...";
    }
    
    /// готов
    void on_ready( )
    {
        std::cout << "ready...";
    }
    
    /// соединение закрыто
    void on_disconnect( )
    {
        std::cout << "disconnect...";
    }
}

```

Опять же это не обязательный код, который можно удалить. 

#### main

В функции main: 

    создается пул потоков, 
    создается клиент, 
    происходит подписка на сигналы клинета, 
    клиент соединяется, 
    при успешном соединении создается канал, делается удаленный вызов и выводится результат

код: 

```cpp

int main( int argc, const char **argv )
{
    /// управлялка потоками. При старте создадим один поток, 
    /// потому что работа клиента асинхронная, несмотря на то, что 
    /// вызов connect не вернет управление до тех пор, пока клиент не станет готов
    /// либо не произойдет ошибка, которая сгенерирует исключение.
    common::thread_pool tp( 1 );

    /// значения по умолчанию
    const char *address = "127.0.0.1";
    unsigned short port = 56560;

    /// настроим так же как и в случае сервера
    if( argc > 2 ) {
        address = argv[1];
        port = boost::lexical_cast<unsigned short>( argv[2] );
    } else if( argc > 1 ) {
        port = boost::lexical_cast<unsigned short>( argv[1] );
    }

    /// попробуем .... 
    try {

        /// создаем экземпляр клиента 
        /// как и в случае со слушателями сервера мы пользуемся функцией-фабрикой,
        /// которая вернет нам vtrc_client_sptr (shared_ptr<vtrc_client>)
        client::vtrc_client_sptr cl =
                         client::vtrc_client::create( tp.get_io_service( ) );

        /// подпишемся на сигналы
        cl->on_connect_connect( on_connect );
        cl->on_ready_connect( on_ready );
        cl->on_disconnect_connect( on_disconnect );

        std::cout <<  "Connecting..." << std::endl;

        /// попытаемся соединиться
        /// в случае успеха на консоль будет выведено 
        /// Connecting...connect...ready...Ok
        cl->connect( address, port );

        std::cout << "Ok" << std::endl;

        /// Создадим канал, который потом отдадим Stub-классу
        vtrc::unique_ptr<common::rpc_channel> channel(cl->create_channel( ));

        /// просто удобный typedef
        typedef howto::hello_service_Stub stub_type;

        /// обертка для вызовов, которая создаст класс
        common::stub_wrapper<stub_type> hello(channel.get( ));

        /// сообщения запроса и ответа
        howto::request_message  req;
        howto::response_message res;

        /// методы с префиксом set_, либо mutable_ создаются генератором-протобуфером
        /// для установки значений 
        req.set_name( "%USERNAME%" );

        /// делаем удаленный вызов. 
        /// если бы hello был экземпляром Stub-класса, 
        /// то вызов выглядел бы так: 
        ///     stub_type hello(channel.get( ));
        ///     ......
        ///     hello.send_helo( NULL, &req, &res, NULL );
        hello.call( &stub_type::send_hello, &req, &res );

        /// вывод результата, который отдал сервер
        std::cout <<  res.hello( ) << std::endl;

    } catch( const std::exception &ex ) {
        std::cerr << "Hello, world failed: " << ex.what( ) << "\n";
    }

    /// остановим потоки и подождем пока они не прекратят работу
    tp.stop( );
    tp.join_all( );

    /// make valgrind happy.
    google::protobuf::ShutdownProtobufLibrary( );

    return 0;
}

```
вот и весь клиент


пробуем: 

сервер: 

    [root@virt2real default]# ./hello_server 0.0.0.0 50000 

клиент:

    % ./hello_client 192.168.3.1 50000 
    Connecting...
    connect...ready...Ok
    Hello %USERNAME% from hello_service_impl::send_hello!
    Your transport name is 'tcp://192.168.3.10:53354'.
    Have a nice day.
    
















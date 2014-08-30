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
    
    /// будем сохранять информацию о клиенте, владеющем этим классом
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

##Client



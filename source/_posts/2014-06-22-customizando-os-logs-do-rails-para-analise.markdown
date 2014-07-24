---
layout: post
title: "Customizando os Logs do Rails para Análise"
subtitle: "Contextualização e formatação para simplificar os filtros e a análise automatizada"
date: 2014-06-22 02:04:51 -0300
comments: true
meta: false
categories: [Ruby, Rails, Log]
---
Formatar os logs da aplicação para simplificar o parsing será nosso principal objetivo nesse post, esta é a primeira etapa de um outro objetivo que é centralizar e analisar programaticamente esses logs para identificar eventos e aplicar ações como indexar, arquivar, notificar (por email/group chat) ou simplesmente delegar para outras ferramentas como bug tracker, dashboard de estatísticas (técnicas e/ou de negócio), etc.
<!--more-->
O log padrão do Rails é feito em múltiplas linhas, o que facilita a leitura direta durante o desenvolvimento mas dificulta o parsing.

A [documentação do Fluentd](http://docs.fluentd.org/articles/in_tail) exemplifica o parsing multiline de um GET padrão do Rails mas basta acontecer qualquer coisa diferente do “normal” (como um warning ou exception com backtrace) que a regex é rapidamente invalidada.
Para superar essa dificuldade vamos logar cada request em uma única linha e além disso adicionar informações para contextualizar o conteúdo de acordo com a nossa necessidade.

Existem diversas [ferramentas para customizar os logs do Rails](https://www.ruby-toolbox.com/categories/Logging), depois de testar algumas escolhi a gem [Lograge](https://github.com/roidrage/lograge) para nos ajudar neste ponto.

A primeira coisa a fazer é instalar e configurar a gem:

``` ruby Gemfile
gem “lograge”
```

``` ruby config/application.rb
config.log_level :info
config.lograge.enabled = true
```

Com isso já temos um log bem mais fácil de parsear:

    method=GET path=/ format=html controller=site action=index status=200 duration=262.28 view=108.93 db=3.84

Vamos ainda extrair mais algumas informações dos nossos objetos “request“ e “session” criando o módulo CustomLog::ControllerHelper:

``` ruby lib/custom_log/controller_helper.rb
module CustomLog
  module ControllerHelper
    extend ActiveSupport::Concern
    included do
      def append_info_to_payload(payload)
        super
        payload[:request] = request
        payload[:session] = session
      end
    end
  end
end
```

Incluiremos esse módulo em nosso ApplicationController:

``` ruby app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include CustomLog::ControllerHelper
end
```

Agora temos a possibilidade de extrair as informações que desejamos, faremos isso criando a classe CustomLog::Subscriber:

``` ruby lib/custom_log/subscriber.rb
module CustomLog
  class Subscriber
    attr_reader :event, :payload
    attr_accessor :data

    def initialize(event)
      @event = event
      @payload = event.payload
      @data = { :time => %Q(‘#{event.time}’) }
    end

    def extract_data
      session_data
      request_data
      params_data

      return data
    end

    private

    def session_data
      session = payload[:session]
      if session
        data.merge!({ :session_id => session[:session_id] })
      end
    end

    def request_data
      request = payload[:request]
      if request
        data.merge!({
          :host => request.host,
          :request_id => request.uuid,
          :remote_ip => request.remote_ip,
          :user_agent => %Q(‘#{request.user_agent}’),
          :referer => request.referer,
          :flash => request.flash.instance_variable_get(‘@flashes’)
        })
      end
    end

    def params_data
      internal_params = %w(controller action format
                           utf8 authenticity_token commit)
      params = payload[:params]
      if params
        data.merge!({ :params => params.except(*internal_params) })
      end
    end
  end
end
```

Configuramos também a gem “lograge” para usar nossa classe:

``` ruby config/environments/development.rb
config.lograge.custom_options = lambda { |event| 
  CustomLog::Subscriber.new(event).extract_data 
}
```

Agora complementamos nosso log com:

    time=’2014-06-15 13:13:47 -0300' session_id=abcde12345 host=app01 request_id=a1b2c3d4e5 remote_ip=179.209.205.143 user_agent=’Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:26.0) Gecko/20100101 Firefox/26.0' referer=/things/new flash={:notice=>”Dados gravados com sucesso”} params={“thing”=>{“name”=>”Teste”}

Além disso vamos incluir também as mensagens de validação do objeto que estamos manipulando, para isso vamos alterar o helper e o subscriber:

``` ruby lib/custom_log/controller_helper.rb
module CustomLog
  module ControllerHelper
    ...
    def validation_messages
      saved_object.errors.to_json unless saved_object.errors.blank?
    rescue
      nil
    end
    def saved_object
      instance_variable_get("@#{controller_name.singularize}")
    end
    ...
    def append_info_to_payload(payload)
      ...
      payload[:validations] = validation_messages
    end
  end
end
```
``` ruby lib/custom_log/subscriber.rb
module CustomLog
  class Subscriber
    def extract_data
      ...
      validations_data
      ...
    end
    ...
    def validations_data
      validations = payload[:validations]
      if validations
        data.merge!({ :validations => validations })
      end
    end
  end
end
```

complementando o log com coisas do tipo:

    flash={:error=>”Não foi possível gravar os dados”} validations={"name":["já está em uso"]}

Em uma aplicação multitenant também é importante ter informações sobre o usuário e o tenant (empresa, conta, etc) com que os requests estão interagindo, para isso alteramos mais uma vez o helper e o subscriber:

``` ruby lib/custom_log/controller_helper.rb
module CustomLog
  module ControllerHelper
    ...
    def custom_log_data(payload)
      payload[:custom_log_data] = {}
    end
    ...
    def append_info_to_payload(payload)
      ...
      custom_log_data(payload)
    end
  end
end
```
``` ruby lib/custom_log/subscriber.rb
module CustomLog
  class Subscriber
    def extract_data
      ...
      custom_data
      ...
    end
    ...
    def custom_data
      custom = payload[:custom_log_data]
      if custom.is_a?(Hash)
        custom.each_pair { |key, value| data[key] = value }
      end
    end
  end
end
```

Como essas informações são específicas de cada aplicação precisamos customizar o método em nosso controller:

``` ruby app/controllers/application_controller.rb
def custom_log_data(payload)
  payload[:custom_log_data] = {
    user_id: current_user.try(:id),
    account_id: current_account.try(:id)
  }
end
```

Dessa forma complementamos nosso log com:

    user_id=987 account_id=123

E para finalizar vamos logar o backtrace completo em caso de exceptions, para isso vamos complementar o [Instrumenter do ActiveSupport](https://github.com/rails/rails/blob/3-2-stable/activesupport/lib/active_support/notifications/instrumenter.rb):

``` ruby config/initializers/log_backtrace.rb
module ActiveSupport
  module Notifications
    class Instrumenter
      attr_reader :id

      def initialize(notifier)
        @id = unique_id
        @notifier = notifier
      end

      def instrument(name, payload={})
        started = Time.now
        begin
          yield
        rescue Exception => e
          payload[:exception] = [e.class.name, e.message]
          payload[:backtrace] = e.backtrace
          raise e
        ensure
          @notifier.publish(name, started, Time.now, @id, payload)
        end
      end
    end
  end
end
```

e alterar novamente nosso subscriber:

``` ruby lib/custom_log/subscriber.rb
module CustomLog
  class Subscriber
    def extract_data
      ...
      exception_data
      ...
    end
    ...
    def exception_data
      exception = payload[:exception]
      backtrace = payload[:backtrace]
      if exception
        data.merge!({
          :exception_class => exception.first,
          :exception_message => exception.last,
          :backtrace => %Q(‘#{Array(backtrace).to_json}’)
        })
      end
    end
  end
end
```

Isso adiciona ao  nosso log informações como:

    exception_class=RuntimeError exception_message=Test backtrace=’[“complete", "backtrace", "array"]'

Com isso atingimos nosso objetivo que é ter o log em uma única linha contextualizado com todas as informações que nos interessam para fazer análise de forma automatizada.

Por precaução fiz um benchmark bem  simples com o antes/depois de nossas customizações e a diferença foi de 0.01 segundo simulando 1000 requests com concorrência de 30, o que acredito ser aceitável para a maioria dos casos.

Veja um [Gist com o conteúdo completo](https://gist.github.com/fernandoalmeida/ef03643fd8d924cc034a) de cada arquivo que criamos aqui.

Na próxima etapa vamos usar um agregador para receber esses logs customizados, parsear  e tomar decisões sobre onde persistir, arquivar, indexar, notificar, disponibilizar para outros serviços (via [PubSub](http://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern), webhook, etc) totalmente independente/desacoplado da aplicação e tolerante a falhas.

**Referências e links relacionados:**

- http://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
- http://gojko.net/2006/12/09/logging-anti-patterns/
- http://blog.mmlac.com/log4r-for-rails/
- http://rubyjunky.com/cleaning-up-rails-4-production-logging.html
- http://www.paperplanes.de/2012/3/14/on-notifications-logsubscribers-and-bringing-sanity-to-rails-logging.html
- http://www.railsonmaui.com//blog/2013/05/08/strategies-for-rails-logging-and-error-handling/
- http://edgeguides.rubyonrails.org/active_support_instrumentation.html

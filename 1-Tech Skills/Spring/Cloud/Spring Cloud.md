# Spring Cloud

* [Spring Cloud Config](#spring-cloud-config)
* [Spring Cloud Service Discovery (Eureka)](#spring-cloud-service-discovery-eureka)
* [Шаблоны устойчивости на стороне клиента](#шаблоны-устойчивости-на-стороне-клиента)
  * [CircuitBreaker](#circuitbreaker)
  * [BulkHead](#bulkhead)
  * [Retry](#retry)
  * [RateLimiter](#ratelimiter)
  * [ThreadLocal и Resilience4J](#threadlocal-и-resilience4j)
* [Spring Cloud Gateway](#gateway)

### Spring cloud config

#### Client:
* Подключение: spring-cloud-starter-config
* `@RefreshScope` обновляет конфиг в тех бинах, где указан, т.е. указывать надо не где main метод, а в конкретном сервисе.
* Указание конфиг сервера – spring.config.import=optional:configserver:<server_address>
* Для обновления бинов и конфигов нужно явно вызвать actuator/refresh
* Загружает тот профиль, что указан в spring.profiles.active

#### Config:
* Подключение: spring-cloud-config-server + @EnableConfigServer
* Native – спец профиль для подключения конфига на локальной машине из папки
* Указать путь к конфигу на локальной машине: spring.cloud.config.server.native.search-locations: file:///<путь>
* 3 слэша(///) после file: специально для Windows
* Boostrap.yaml устаревший файл для конфига. Теперь все лежит в application.yaml
* Чтобы подключить boostrap файл, надо добавить зависимость boostrap-starter
* Git конфиг: git.uri – адрес git репозитория, можно ссылаться на удаленный репозиторий, например GitHub
* git.search-paths – путь внутри репозиторий, можно делать в видео параметров {application-name}
* default-label – ветка по умолчанию, в которой надо искать. Без параметра используется main, потом master
* clone-on-start – клонирует репозиторий при запуске
* force-pull – забирает данные, если локальная копия изменилась
* git аутентификация – по умолчанию используется ssh для удаленного репозитория. Можно использоваться username и password. Либо указать параметры hostKeyAlgorithm и privateKey с приватным ключом и алгоритмом для аутентификации по SSH

#### Actuator:
* `/refresh` по умолчанию недоступен
* Включены по умолчанию все эндпоинты, но expose(отображаются) только info и health
* Для отображения(expose) надо задать management.endpoints.web.exposure.include=* или «*» для yaml или перечислить конкретные эндпоинты
* `/refresh` это POST запрос

#### Spring Config основные параметры
```yaml
spring:
  application:
    name: # имя приложения
  profiles:
    active: # профиль для текущего конфига
  cloud:
    config:
      server:
        native:
          search-locations: # Расположение папки с конфигами на локальной машине для Config Server
        git:
          uri: # Адрес гит репозитория с конфигом
          search-paths: # Адрес внутри гит репозитория
          default-label: # Ветка, в которой нужно искать, main по умолчанию, потом master
          clone-on-start: # Клонирование репозитория при запуске
          force-pull:  # Обновление репозитория, если локальный изменился
server:
  port: # порт приложения
```

#### Spring основные аннотации
* `@Value("${param.name}")` - Подставляет значение из конфиг файла
* `@RestController` - Указание, что это REST контроллер
* `@RefreshScope` - Обновление бинов и конфигов динамически после вызова POST actuator/refresh
* `@RequestMapping("/pets")` - Указание эдпоинта для контроллера
* `@GetMapping` - GET контроллер, сокращение от RequestMapping (GET)
* `@SpringBootApplication` - Главная аннотация для Spring Boot приложения
* `@EnableConfigServer` - Включение конфиг сервера

### Spring Cloud Service Discovery (Eureka)

#### Eureka Server
* Добавить зависимость spring-cloud-starter-netflix-eureka-server
* Аннотировать конфиг `@EnableEurekaServer`
* По дефолту регистрирует сервисы в течение 30 секунд
* После отключения сервиса будет пытаться проверить его healthcheck и примерно через 4-5 минут отрубит его.
* Evicting services – процесс отключения сервисов из Eureka
* self-preservation – процесс, когда Eureka перестает отключать сервисы, в силу слабой сети, когда сервис жив, но не могут пока до него достучаться, пока не пройдет порог(renewal rate)

#### Eureka Server Self Preservation Config
```yaml
server:
  port: # порт сервиса Eureka, деф. 8761
eureka: # блок для конфига Eureka
  client: # конфиг клиента
    registerWithEureka: # Регистрация в Eureka (на сервере выключать, false)
    fetchRegistry: # Копировать к себе копию реестра сервисов
  server: # конфиг сервера
    enable-self-preservation: # Вкл/выкл самосохранение, дефолт true
    expected-client-renewal-interval-seconds: # Ожидание Eureka, через какое время ждет ответа сервиса, дефолт 30сек
    eviction-interval-timer-in-ms: # Джоба, которая запускается, по дефолту раз в 60 сек, и удаляет неработающие сервисы
    renewal-percent-threshold: # Коэффициент, на основе которого вычисляет кол-во ответов от сервиса к Eureka, дефолт 0.85
    renewal-threshold-update-interval-ms: # Джоба для подсчета общего кол-ва ответов(heartbeat) сервисов, дефолт 15 минут
  instance: # конфиг для зарегистрированного сервиса
    lease-expiration-duration-in-seconds: # Время с последнего ответа от сервиса, в течение которого Eureka ждет, прежде, чем удалить сервис из registry, дефолт 90сек
```

#### Eureka client
* Зависимость `spring-cloud-starter-netflix-eureka-client`, опционально `open-feign` для запросов через название сервисов автоматическим load balancer
* Никаких спец аннотация не нужно, сервис автоматически попытается зарегистрироваться в Eureka
* ID сервиса(по которому он регистрируется) – spring.application.name
* Прописать параметры в конфиге eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka – адрес, по которому лежит Eureka и сервис там зарегистрируется.
* eureka.instance.preferIpAddress: true/false – регистрация по IP адресу или названию сервиса. Предпочтительней по IP, т.к. сервисы обычно будут в докере и его название всегда разное и придется контролировать везде указанное название


#### Feign
* Тулза от Netflix, которая помогает делать REST запросы
* Определить интерфейс
* Аннотировать его @FeignClient(name=<имя вызываемого сервиса>). Дополнительно можно указать URL.
* Объявить обычную функцию в интерфейсе и пометить, как обычный Get- PostMapping
* Сделать @Autowired в нужном сервисе и вызвать метод
* Feign сам сделает обертку из интерфейса

### Шаблоны устойчивости на стороне клиента
* Основные причины использования таких шаблонов:
  * Производительность сервиса может снижать постепенно, а не сразу обрубаться. Либо работать нестабильно.
  * Реакция на частичную деградацию
* _Resilience4J_ – библиотека с реализациями шаблонов устойчивости на стороне клиента

Шаблоны
* _Load Balancer_ – балансировка нагрузки
* _Circuit Breaker (размыкатель цепи)_ – разрывает связь, если сервис долго не отвечает, может использовать альтернативный метод для вызова
* _BulkHeads(герметичные отсеки)_ - изолирование пула потоков для работы с сервисами, чтобы останавливать работу с другими сервисами при медленной работе одного
* _Fallback(резервная реализация)_ – альтернативный вызов, если сервис не работает
* _Rate Limit_ - ограничение кол-ва запросов в период времени
* _Time Limit_ - ограничение времени на запросы
* _Retry_ - повтор запроса
* Очередность вызовов(справа налево) – **Bulkhead** > **Time Limiter** > **Rate Limiter** > **Circuit Breaker** > **Retry**


#### CircuitBreaker
* Зависимость _spring-cloud-starter-circuitbreaker-resilience4j_
* `@CircuitBreaker(name="", fallback="")` – аннотация для метода(любого) или класса для использования декоратора размыкателя цепи.
* Fallback метод должен иметь аналогичную сигнатуру и находится в том же классе
* Состояния размыкателя
  * OPEN – все запросы проходят
  * HALF_OPEN – часть запросов проходит для теста сервиса
  * CLOSED – запросы не проходят
  * *DISABLED – доп состояние, все запросы всегда проходят, выходить из него принудительно
  * *FORCED_OPEN – доп состояние, все запросы всегда не проходят, выходить из него принудительно
* Использует кольцевой битой буффер(1 – неуспешные запросы, 0 - успешные). Только при полном заполнении, сможет рассчитать коэффициент отказа и перейти в другое состояние

##### Конфиг

```yaml
resilience4j.circuitbreaker: # конфиг для шаблона размыкателя
  instances:
    given-name: # название из аннотации, любое значение
      registerHealthIndicator: # true/false, отображение в health актуатора 
      ringBufferSizeInClosedState: # буффер кольца в закрытом состоянии, 100 дефолт 
      ringBufferSizeInHalfOpenState: # буффер кольца в наполовину открытом состоянии, 10 дефолт
      waitDurationInOpenState: # продолжительность ожидание в открытом состоянии(10s пример), 60s дефолт
      failureRateThreshold: # порог частоты отказов в открытом состоянии, 50 дефолт
      recordExceptions: # исключения, которые расцениваются как сбои(список через ‘-‘ ), по умолчанию все ошибки при запросе
        - java.test.package.Exception # пример значения
```

#### BulkHead
* Шаблон герметичных отсеков
* Используется для разграничения использования поток, чтобы медленные службы не занимали все потоки, а остальные тем временем не простаивали
* Использует 2 типа SEMAPHOR и THREADPOOL
* SEMAPHOR – ограничение на общее количество вызовов, после чего все запросы отклоняются. Использоваться преимущественно для равномерных запросов к службе.
* THREADPOOL – для каждой службы используется свой пул потоков. Использовать лучше, если частота и время запросов непонятны.
* Используется совместно с CircuitBreaker (? Точно ли, в книге так)
* @BulkHead(name=<имя>, fallback=<резервный метод>, type=<тип, по умолчанию SEMAPHOR>)

##### Конфиг

```yaml
resilience4j.bulkhead: # конфиг для шаблона герметичных отсеков
  instances:
    given-name: # название из аннотации, любое значение
      # Для SEMAPHOR
      maxWaitDuration: # 10ms – макс. время блокировки потока, деф. 0
      maxConcurrentCalls: # 20 – макc. кол-во одновременных вызовов, деф. 25
      # Для THREADPOOL
      maxThreadPoolSize: # 21 – макс. Кол-во потоков, деф. кол-во ядер ПК
      coreThreadPoolSize: # 2 – размер основного пула, деф. кол-во ядер
      queueCapacity: # 10 – размер очереди потоков(по достижении потолка пула), деф. 100
      keepAliveDuration: # 20ms – макс. Время, в течение которого простаивающие потоки ждут новых заданий перед завершением. Учитывается, когда кол-во потоков превышает базовый размер(coreThreadPoolSize).
```

#### Retry
* Используется для повторного вызова
* `@Retry(name = "petsserviceretry", fallbackMethod = "getDefaultFood")`

##### Конфиг
```yaml
resilience4j.retry: # конфиг для шаблона повторных запросов
  instances:
    given-name: # название из аннотации
      maxRetryAttempts: 5 # Макс. кол-во повторных попыток, деф. 3
      waitDuration: 1000 # Макс. кол-во повторных попыток, деф. 500мс
      retry-exceptions: # Ошибки для вызова повтора, деф. пустой список
        - java.util.concurrent.TimeOutException # пример
      intervalFunction: # функция, вычисляющая время ожидания после сбоя
      retryOnResultPredicate: # предикат, оценивающий по полученному результату, стоит ли вызывать еще раз. true, если стоит
      retryOnExceptionPredicate: # аналогично выше, только по типу ошибки
      ignoreExceptions: # список исключений, для которых не нужен retry, деф. пустой список
```

#### RateLimiter
* Ограничивает количество вызовов в определенный период времени
* Отличие от BulkHead в том, что BulkHead это количество одновременных вызовов(например, 10 concurrent запросов), а RateLimiter это общее количество вызовов за определенный период(например, 20 запросов в течение 1 секунды)
* Есть 2 реализации – SemaphorBasedRateLimiter и AtomicRateLimiter
* SemaphorBasedRateLimiter – самый простой
* `@RateLimiter(name = "petsserviceratelimiter", fallbackMethod = "getDefaultFood")`

##### Конфиг
```yaml
resilience4j.ratelimiter:
  instances:
    petsserviceretry: # название из аннотации
      timeDuration: 100ms # время, в течение которого поток ждет разрешения
      limitRefreshPeriod: 500 # период, в течение которого разрешено определенное кол-во вызовов limitForPeriod
      limitForPeriod: 5 # кол-во вызовов, разрешенных в течение периода времени limitRefreshPeriod
```

#### ThreadLocal и Resilience4J
* Шаблоны и аннотации декорируют классы и есть много одновременных потоков, которыми управляют в них
* Чтобы использовать общий контекст, например идентификатор корреляции или что-то, что должно быть доступно из всех потоков, то можно использовать ThreadLocal переменную.
* Для ThreadLocal переменной можно создать переменную ContextHolder(пример) и хранить нужную переменную в этом классе, закрыв все методы редактирования, своего рода Singleton.


#### Gateway

* Используется для перенаправления запросов
* Единая точка входа в приложение
* Можно менять урлы сервисов, но клиент всегда будет обращаться к урлу Gateway
* Может быть зарегистрирован в Eureka
* Зависимость org.springframework.cloud:spring-cloud-starter-gateway
* /actuator/gateway/routes – отображение всех маршрутов, которые определены в Gateway
* Маршруты можно определить автоматически(через ту же Eureka), либо руками замаппить
* Для отображения в actuator нужно добавить в конфиг
```yaml
management:
  endpoint:
    gateway:
      enabled: true
```
* Для отображения урлов нужно включить 
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```
* Можно определить интерсепторы до и после запроса для обработки запроса
* GlobalFilter – интерфейс для определения pre- фильтра
* AbstractGatewayFilterFactory – класс фабрики для создания своего фильтра, для конкретного маршрута

Пример pre- фильтра
```java
@Component
@Order(1)
public class CommonFilter implements GlobalFilter {

    private static final Logger logger = LoggerFactory.getLogger(CommonFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        String CORRELATION_ID = "pts-correlation-id";
        if (headers.get(CORRELATION_ID) != null) {
            String correlationId = headers.get(CORRELATION_ID).stream().findFirst().get();
            logger.info("Found " + CORRELATION_ID + "=" + correlationId);
        } else {
            String newId = UUID.randomUUID().toString();
            exchange = exchange.mutate()
                    .request(exchange.getRequest().mutate().header(CORRELATION_ID, newId).build()).build();
            logger.info("Created new " + CORRELATION_ID + "=" + newId);
        }

        return chain.filter(exchange);
    }
}
```

Пример post- фильтра
```java
@Configuration
public class PostFilter {

    private static final Logger logger = LoggerFactory.getLogger(PostFilter.class);

    @Bean
    public GlobalFilter postFilter() {
        return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() -> { logger.info("Post filter log"); }));
    }
}
```

##### Конфиг
```yaml
spring:
  application:
    name: pets-gateway-server
  cloud:
    gateway:
      routes:
        - id: pets-service # просто ID, можно указать любой
          uri: lb://pets-service # lb означает, что через load balancer, uri - путь конечного сервиса, такой поиск обычно через Eureka
          predicates: # блок предикатов
            - Path=/pets/** # какая маска пути для выборки нужного
          filters: # блок фильтров, которые изменяют запрос
            - RewritePath=/pets/(?<path>.*), /$\{path} # переписывание запроса, замена из 1 аргумента на аргумент 2
        - id: pets-service # 
          uri: http://localhost:8010 # статичный адрес, ручное определение
          predicates: # 
            - Path=/pets/** # 
          filters: # 
            - RewritePath=/pets/(?<path>.*), /$\{path} # 
      discovery.locator: # включение автоматического поиска через Eureka по spring.application.name
        enabled: true # 
        lowerCaseServiceId: true #
```





























	


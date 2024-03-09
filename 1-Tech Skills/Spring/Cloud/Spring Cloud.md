# Spring Cloud

* [Spring cloud config](#spring-cloud-config)
* [Spring Config основные параметры](#spring-config-основные-параметры)

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

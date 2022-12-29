# kafka_docker_scripts

Краткое описание действий по поиску и запуска дистрибутива kafka в ubuntu из docker при использовании технологии wsl:

У меня изначально не получалось запустить докер в ubuntu, поскольку требовалось настроить конфигурацию. 

Я нашел актуальный образ ubuntu 20.04

Но выяснилось, что для того, чтобы запустить докер в убунте из-под wsl 2, потребовалось активировать конфигурацию wsl2 в настройках docker в самом windows, 
и применить команду трансформации в power shell:   wsl --set-version Ubuntu 2 

Для разворачивания Kafka в Ubuntu проверяем, что изменение версии корректно, и только после этого запускаем Ubuntu:

PS C:\Users\vyuivanov> wsl -l -v
PS C:\Users\vyuivanov> wsl --distribution Ubuntu --user vi

Находясь в Ubuntu, я нашел два образа kafka, ubuntu/kafka и bitnami/kafka
но хорошо сработал именно последний:


Теперь, когда этот образ есть, можем выполнить его запуск:
Посмотрим теперь на запущенные контейнеры docker:

Для наглядности: это bitnami/kafka:3.3 (ID d5a7642e45d0) и  
                                    bitnami/zookeeper:3.8(ID 31df1c2b4804), их имена и порты:

vi-kafka-1          0.0.0.0:9092->9092/tcp, :::9092->9092/tcp
vi-zookeeper-1  2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 8080/tcp\\

В конфигурации этих контейнеров прописано следующее:
vi@...:~$ cat docker-compose.yml
version: "2"

services:
  zookeeper:
    image: docker.io/bitnami/zookeeper:3.8
    ports:
      - "2181:2181"
    volumes:
      - "zookeeper_data:/bitnami"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
  kafka:
    image: docker.io/bitnami/kafka:3.3
    ports:
      - "9092:9092"
    volumes:
      - "kafka_data:/bitnami"
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper

volumes:
  zookeeper_data:
    driver: local
  kafka_data:
    driver: local


Текущие директории в них установлены как корневые:
vi@...:~$  docker exec -it vi-kafka-1 pwd
/
vi@...:~$ docker exec -it vi-zookeeper-1  pwd
/

Инструкции по работе с этим контейнером находятся здесь:
https://hub.docker.com/r/bitnami/kafka

Поработаем с ним. Для примера мы создадим экземпляр клиента Apache Kafka, который будет подключаться к экземпляру сервера, работающему в той же сети докеров, что и 
клиент.

Шаг 1: Создаем сеть
Шаг 2: Запускаем экземпляр сервера Zookeeper
Шаг 3. Запускаем экземпляр клиента Apache Kafka

На текущий момент, при выполнении последней инструкции мы получили предупреждение:

Такая проблема описана здесь: 
https://stackoverflow.com/questions/66590997/kafka-server-issue-in-docker

А именно, мы узнаем следующее:
Вы пытаетесь связаться с kafka:9092, но docker compose сгенерировал контейнер kafka_1, поэтому разрешение имени отсутствует.

Docker дает внутренний IP-адрес вашим контейнерам и использует их имя контейнера для создания внутреннего DNS в этой сети (со встроенным DNS-сервером).
Docker Compose не изменяет ваши переменные среды, чтобы они соответствовали имени, которое он дает вашему контейнеру.
Вы должны использовать «container_name: kafka» в описании вашего контейнера, чтобы получить статическое имя контейнера.

Всё же, попробуем теперь создать топик:
Мы видим, что топик “orders” появился:
Далее можно будет попробовать организовать отправку сообщений на этот топик.

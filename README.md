# kafka_docker_scripts

Описание действий по поиску и запуска дистрибутива Apache Kafka в Ubuntu из Docker при использовании технологии WSL 2(Windows Subsystem for Linux):

Используем образ Ubuntu 20.04

![ubuntu 20.04](/images/picture.png)

Для использования *WSL 2* нужно активировать эту конфигурацию в настройках Docker в самом Windows 10, 
и применить команду трансформации образа Ubuntu в Power Shell:   
```wsl --set-version Ubuntu 2``` 

Для разворачивания Kafka в Ubuntu проверяем, что изменение версии корректно, и запускаем Ubuntu: 
```PS C:\Users\user> wsl -l -v```
```
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
```
А теперь выполяем запуск, пользователь, для примера, "vi":
```
PS C:\Users\vyuivanov> wsl --distribution Ubuntu --user vi
```
С этого момента работем уже в консоли Ubuntu, которая должна открыться после предыдущей команды.
Перед загрузкой образа kafka, которую мы выполним ниже, необходимо активировать службу docker, если этого ещё не сделано:
```
> sudo service docker start
```
Теперь можно подтягивать образы docker. Есть несколько образов Kafka, это "ubuntu/kafka" и "bitnami/kafka"
с наименьшими измененями конфигурации перед запуском хорошо работает именно последний, "bitnami", который мы загрузим командой pull:
``` 
> docker pull bitnami/kafka
docker.io/bitnami/kafka:latest
```

Теперь, когда этот образ есть, можем выполнить его запуск:
```
   > curl -sSL https://raw.githubusercontent.com/bitnami/containers/main/bitnami/kafka/docker-compose.yml > docker-compose.yml
   > docker-compose up -d
```
Результат запуска такой:
```[+] Running 4/4
 ⠿ kafka Pulled                                                                                                                                                                                              37.4s
 ⠿ zookeeper Pulled                                                                                                                                                                                          69.0s
   ⠿ f8c1c832ce65 Already exists                                                                                                                                                                              0.0s
   ⠿ bfe5e86e413f Pull complete                                                                                                                                                                              31.0s
[+] Running 5/5
 ⠿ Network vi_default          Created                                                                                                                                                                        0.7s
 ⠿ Volume "vi_zookeeper_data"  Created                                                                                                                                                                        0.0s
 ⠿ Volume "vi_kafka_data"      Created                                                                                                                                                                        0.0s
 ⠿ Container vi-zookeeper-1    Started                                                                                                                                                                        1.6s
 ⠿ Container vi-kafka-1        Started                                                                                                                                                                        2.6s
```

Посмотрим теперь на запущенные контейнеры Docker:
```docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS          PORTS                                                                     NAMES
d5a7642e45d0   bitnami/kafka:3.3       "/opt/bitnami/script…"   41 minutes ago   Up 41 minutes   0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                 vi-kafka-1
31df1c2b4804   bitnami/zookeeper:3.8   "/opt/bitnami/script…"   41 minutes ago   Up 41 minutes   2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 8080/tcp   vi-zookeeper-1
```

В конфигурации этих контейнеров прописано следующее:
```
> cat docker-compose.yml
```
```
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
```

Инструкции по работе с этим контейнером находятся здесь:
https://hub.docker.com/r/bitnami/kafka

Поработаем с ним. Для примера мы создадим экземпляр клиента Apache Kafka, который будет подключаться к экземпляру сервера, работающему в той же сети докеров, что и 
клиент.

Шаг 1: Создаем сеть
```
> docker network create app-tier --driver bridge
```

Шаг 2: Запускаем экземпляр сервера Zookeeper
```
> docker run -d --name kafka-server --network app-tier -e ALLOW_PLAINTEXT_LISTENER=yes -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 bitnami/kafka:latest
```

Шаг 3. Запускаем экземпляр клиента Apache Kafka
```
> docker run -it --rm --network app-tier -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 bitnami/kafka:latest kafka-topics.sh --list  --bootstrap-server kafka-server:9092
```
Получаем:
```
kafka 16:34:13.37
kafka 16:34:13.37 Welcome to the Bitnami kafka container
kafka 16:34:13.37 Subscribe to project updates by watching https://github.com/bitnami/containers
kafka 16:34:13.37 Submit issues and feature requests at https://github.com/bitnami/containers/issues
```

А теперь создаём топик:
```
> docker exec vi-kafka-1 kafka-topics.sh --bootstrap-server localhost:9092 --create --topic orders --partitions 1 --replication-factor 1
Created topic orders.
```
Проверяем, что Kafka видит этот топик:
```
> docker exec vi-kafka-1 kafka-topics.sh --bootstrap-server localhost:9092 --list
orders
```

Это означает, что топик "orders" появился.

-----------start-----------
application properties for config server
========================================
spring cloud bus config
=======================
management.endpoints.web.exposure.include= refresh, bus-refresh
spring.application.name= config-server
server.port= 3100
git related external configuration
==================================
spring.profiles.active= git
spring.cloud.config.server.git.uri= https://github.com/nagababuballa/Spring-External-Configuration
spring.cloud.config.server.git.default-label= main
spring.cloud.config.server.git.search-paths= .
rabbit mq messaging configuration
=================================
spring.rabbitmq.host= localhost
spring.rabbitmq.port= 5672
spring.rabbitmq.username= guest
spring.rabbitmq.password= guest
logging configuration
=====================
logging.config= classpath:logback-spring.xml
------------End------------

-----------start-----------
application properties for discovery server(Eureka)
spring.application.name= discovery-service
spring.config.import= optional:configserver:http://localhost:3100
------------End------------

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
@EnableEurekaServer

Centralized Logging Configuration
=================================
1)each and every microservice write the log to a file that contains the log information
logging.file.name=/var/log/microservice-name.log
logging.pattern.file={"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSSZ}", "level":"%p", "thread":"%t", "logger":"%c", "message":"%m"}
2)Configure file beats in each micro service which is responsible for monitoring the log file and push it to logstash
filebeat.yml
============
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log  # Path to log files
    fields:
      application: "microservice-name"  # Replace with the name of the service
    fields_under_root: true
    multiline:
      pattern: '^{'
      negate: true
      match: after
output.logstash:
  hosts: ["${logstash-server-ip}:5044"]  # Replace with your Logstash server IP
Log stash
==========
logstash-filebeat.conf
----------------------
input {
  beats {
    port => 5044
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"] # Elasticsearch endpoint
    index => "microservices-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
Configure Elastic search and Kibana
Go to Kibana cosole and under indexes lookup for your index 
and see the information about your log file

Distributed Tracing
====================
we are achieving it with the help of Zipkin
-- Micrometer Tracing Bridge --
<dependency>
	<groupId>io.micrometer</groupId>
	<artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
-- Brave Tracer for Zipkin --
<dependency>
	<groupId>io.zipkin.brave</groupId>
	<artifactId>brave</artifactId>
</dependency>
-- Zipkin Reporter for sending traces --
<dependency>
	<groupId>io.zipkin.reporter2</groupId>
	<artifactId>zipkin-reporter-brave</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

application.properties
======================
# Enable Micrometer tracing
management.tracing.enabled=true

# Configure the Zipkin endpoint
management.zipkin.tracing.enabled=true
management.zipkin.tracing.endpoint=http://<zipkin-server-ip>:9411/api/v2/spans

# Enable propagation of trace IDs
spring.sleuth.propagation.type=B3

when making REST calls between microservices, Spring will automatically include tracing
headers (like X-B3-TraceId and X-B3-SpanId) in the requests.

Circuit Breaker
---------------
Inorder to avoid cascading failures
it is the specification and implementation is provided by number of techniques like Netflix Hystrix , Resillence 4J and so on
Since Netflix Hystrix is outdated and it is in maintanence we are using Resillence 4j

dependencies
------------
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-actuator</artifactId>
</dependency>
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

Properties
----------
management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

#Resilinece4j Properties
resilience4j.circuitbreaker.instances.order.registerHealthIndicator=true   ---> able to see those states like open,half-open,close etc.
resilience4j.circuitbreaker.instances.order.event-consumer-buffer-size=10  ---> 10 calls
resilience4j.circuitbreaker.instances.order.slidingWindowType=COUNT_BASED  ---> calls are based on count
resilience4j.circuitbreaker.instances.order.slidingWindowSize=5            ---> check for 5 continuous failures
resilience4j.circuitbreaker.instances.order.failureRateThreshold=50        ---> when thresold becomes 50% (5/10*100)
resilience4j.circuitbreaker.instances.order.waitDurationInOpenState=5s     ---> wait for 5 seconds when it is in half open state and again triggers the calls
resilience4j.circuitbreaker.instances.order.permittedNumberOfCallsInHalfOpenState=3  ---> sends three calls to check the service is avilable or not
resilience4j.circuitbreaker.instances.order.automaticTransitionFromOpenToHalfOpenEnabled=true  ---> Half open to open when fails

#Resilience4J Timeout Properties
resilience4j.timelimiter.instances.order.timeout-duration=3s ---> if service call takes beyond 3s it will throw timeout exception

#Resilience4J Retry Properties
resilience4j.retry.instances.order.max-attempts=3     
resilience4j.retry.instances.order.wait-duration=5s

@PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @CircuitBreaker(name = "order", fallbackMethod = "fallbackMethod")
    @TimeLimiter(name = "order")
    @Retry(name = "order")
    public CompletableFuture<String> placeOrder(@RequestBody OrderRequest orderRequest) {
        return CompletableFuture.supplyAsync(() -> orderService.placeOrder(orderRequest));
    }
    public CompletableFuture<String> fallbackMethod(OrderRequest orderRequest, RuntimeException runtimeException) {
        return CompletableFuture.supplyAsync(() -> "Oops! Something went wrong, please order after some time!");
    }

Kafka
=====
Distributed Messaging system

Topic Creation
--------------
@Bean
    public NewTopic orderTopic() {
        return TopicBuilder
                .name("order-topic")
                .build();
    }
    
Producer configuration
----------------------
spring.kafka.producer.bootstrap-servers= localhost:9092  ----> Kafka server ip address and port
spring.kafka.producer.key-serializer= org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer= org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.producer.properties.spring.json.type.mapping= orderConfirmation:com.alibou.ecommerce.kafka.OrderConfirmation

using kafka template to send the payload over topic
---------------------------------------------------
public void sendOrderConfirmation(OrderConfirmation orderConfirmation) {
        log.info("Sending order confirmation");
        Message<OrderConfirmation> message = MessageBuilder
                .withPayload(orderConfirmation)
                .setHeader(TOPIC, "order-topic")
                .build();
        kafkaTemplate.send(message);
    }

Consumer Configuration
----------------------
spring.kafka.consumer.bootstrap-servers: localhost:9092
spring.kafka.consumer.group-id: paymentGroup,orderGroup
spring.kafka.consumer.auto-offset-reset: earliest
spring.kafka.consumer.key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages: '*'
spring.kafka.consumer.properties.spring.json.type.mapping:orderConfirmation:com.alibou.ecommerce.kafka.order.OrderConfirmation

Receving the Payload from kafka topic using Listener
----------------------------------------------------
@KafkaListener(topics = "order-topic", groupId = "order-group")
@RetryableTopic(attempts = "3")
public void consumeOrderConfirmationNotifications(OrderConfirmation orderConfirmation,@Header(KafkaHeaders.RECEIVED_TOPIC) String topic, @Header(KafkaHeaders.OFFSET) long offset) throws MessagingException {
        log.info("Received: {} from {} offset {}", new ObjectMapper().writeValueAsString(orderConfirmation), topic, offset);
    }
    
@DltHandler 
public void listenDLT(OrderConfirmation orderConfirmation, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic, @Header(KafkaHeaders.OFFSET) long offset) {
  log.info("DLT Received : {} , from {} , offset {}",new ObjectMapper().writeValueAsString(orderConfirmation),topic,offset);
}

Monitoring
==========
we are using prometeus and grafana for monitoring all the micro services
1)Actuator is used fro exposing all the metrics like jvm metrics and all other metrics
2)Promotheus polls micro services for every predefined interval of time
  uses an in memory database to store these metrics
3)Grafana is used for visualization purpose by collecting these metrics from Promotheus
dependencies
------------
add these dependencies in all our micro services
<dependency>
   <groupId>io.micrometer</groupId>
   <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
properties
----------
we need to add the prometheus actuator end point
# Actuator Prometheus Endpoint
management.endpoints.web.exposure.include= prometheus

Install Promotheus and configure it like below
global:
  scrape_interval:     10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: 'product_service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['product-service:8080']--->Hostname:port
        labels:
          application: 'Product Service Application'
  - job_name: 'order_service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8080']
        labels:
          application: 'Order Service Application'
  - job_name: 'inventory_service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['inventory-service:8080']
        labels:
          application: 'Inventory Service Application'
  - job_name: 'notification_service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['notification-service:8080']
        labels:
          application: 'Notification Service Application'

Docker compose setup:
----------------------
version: '3.7'
services:
  ## Postgres Docker Compose Config
  postgres-order:
    container_name: postgres-order
    image: postgres
    environment:
      POSTGRES_DB: order-service
      POSTGRES_USER: ptechie
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./postgres-order:/data/postgres
    expose:
      - "5431"
    ports:
      - "5431:5431"
    command: -p 5431
    restart: always

  postgres-inventory:
    container_name: postgres-inventory
    image: postgres
    environment:
      POSTGRES_DB: inventory-service
      POSTGRES_USER: ptechie
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./postgres-inventory:/data/postgres
    ports:
      - "5432:5432"
    restart: always

  ## Mongo Docker Compose Config
  mongo:
    container_name: mongo
    image: mongo:4.4.14-rc0-focal
    restart: always
    ports:
      - "27017:27017"
    expose:
      - "27017"
    volumes:
      - ./mongo-data:/data/db

  ## Keycloak Config with Mysql database
  keycloak-mysql:
    container_name: keycloak-mysql
    image: mysql:5.7
    volumes:
      - ./mysql_keycloak_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: keycloak
      MYSQL_USER: keycloak
      MYSQL_PASSWORD: password

  keycloak:
    container_name: keycloak
    image: quay.io/keycloak/keycloak:18.0.0
    command: [ "start-dev", "--import-realm" ]
    environment:
      DB_VENDOR: MYSQL
      DB_ADDR: mysql
      DB_DATABASE: keycloak
      DB_USER: keycloak
      DB_PASSWORD: password
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
    volumes:
      - ./realms/:/opt/keycloak/data/import/
    depends_on:
      - keycloak-mysql

## Zookeeper Config for Kafka Server
  zookeeper:
    image: confluentinc/cp-zookeeper:7.0.1
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:7.0.1
    container_name: broker
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://broker:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  ## Zipkin Configuration for Distributed Tracing
  zipkin:
    image: openzipkin/zipkin
    container_name: zipkin
    ports:
      - "9411:9411"

  ## Eureka Server one of the micro service
  discovery-server:
    image: microservices/discovery-server:latest
    container_name: discovery-server
    ports:
      - "8761:8761"
    depends_on:
      - zipkin

  api-gateway:
    image: microservices/api-gateway:latest
    container_name: api-gateway
    ports:
      - "8181:8080"
    expose:
      - "8181"
    environment:
      - LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_SECURITY= TRACE
    depends_on:
      - zipkin
      - discovery-server
      - keycloak

  ## Product-Service Docker Compose Config
  product-service:
    container_name: product-service
    image: microservices-tutorial/product-service:latest
    depends_on:
      - mongo
      - discovery-server
      - api-gateway

  ## Order-Service Docker Compose Config
  order-service:
    container_name: order-service
    image: microservices-tutorial/order-service:latest
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-order:5431/order-service
    depends_on:
      - postgres-order
      - broker
      - zipkin
      - discovery-server
      - api-gateway

  ## Inventory-Service Docker Compose Config
  inventory-service:
    container_name: inventory-service
    image: microservices-tutorial/inventory-service:latest
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-inventory:5432/inventory-service
    depends_on:
      - postgres-inventory
      - discovery-server
      - api-gateway

  ## Notification-Service Docker Compose Config
  notification-service:
    container_name: notification-service
    image: microservices-tutorial/notification-service:latest
    depends_on:
      - zipkin
      - broker
      - discovery-server
      - api-gateway

  ## Prometheus
  prometheus:
    image: prom/prometheus:v2.37.1
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml-->copying to docker /etc/prometheus/prometheus.yml
    depends_on:
      - product-service
      - inventory-service
      - order-service
      - notification-service

  grafana:
    image: grafana/grafana-oss:8.5.2
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    links:
      - prometheus:prometheus
    volumes:
      - ./grafana:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=password

Since we are using Docker for  Grafana it is installed at the time of starting the container
add datasource---->Time series databases ----> Promotheus --->name it as any (Prometheus Micro Service)
http url ---> http://prometheus:9090  ----> Here prometheus is the name of a container
Save and test

Creating dashboard
==================
Grafana_Dashboard.json
----------------------
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "Prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "__elements": [],
  "__requires": [
    {
      "type": "panel",
      "id": "gauge",
      "name": "Gauge",
      "version": ""
    },
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "8.5.2"
    },
    {
      "type": "panel",
      "id": "graph",
      "name": "Graph (old)",
      "version": ""
    },
    {
      "type": "datasource",
      "id": "prometheus",
      "name": "Prometheus",
      "version": "1.0.0"
    },
    {
      "type": "panel",
      "id": "stat",
      "name": "Stat",
      "version": ""
    }
  ],
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": {
          "limit": 100,
          "matchAny": false,
          "tags": [],
          "type": "dashboard"
        },
        "type": "dashboard"
      }
    ]
  },
  "description": "Dashboard for Spring Boot2 Statistics(by micrometer-prometheus).",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "gnetId": 6756,
  "graphTooltip": 0,
  "id": null,
  "iteration": 1654853269928,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 54,
      "panels": [],
      "title": "Basic Statistics",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "decimals": 1,
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "s"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 6,
        "x": 0,
        "y": 1
      },
      "id": 52,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "process_uptime_seconds{application=\"$application\", instance=\"$instance\"}",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "",
          "metric": "",
          "refId": "A",
          "step": 14400,
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Uptime",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "decimals": 1,
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "rgba(50, 172, 45, 0.97)",
                "value": null
              },
              {
                "color": "rgba(237, 129, 40, 0.89)",
                "value": 70
              },
              {
                "color": "rgba(245, 54, 54, 0.9)",
                "value": 90
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 6,
        "w": 5,
        "x": 6,
        "y": 1
      },
      "id": 58,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "sum(jvm_memory_used_bytes{application=\"$application\", instance=\"$instance\", area=\"heap\"})*100/sum(jvm_memory_max_bytes{application=\"$application\",instance=\"$instance\", area=\"heap\"})",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "",
          "refId": "A",
          "step": 14400,
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Heap Used",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "decimals": 1,
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            },
            {
              "options": {
                "from": -1e+32,
                "result": {
                  "text": "N/A"
                },
                "to": 0
              },
              "type": "range"
            }
          ],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "rgba(50, 172, 45, 0.97)",
                "value": null
              },
              {
                "color": "rgba(237, 129, 40, 0.89)",
                "value": 70
              },
              {
                "color": "rgba(245, 54, 54, 0.9)",
                "value": 90
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 6,
        "w": 5,
        "x": 11,
        "y": 1
      },
      "id": 60,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "sum(jvm_memory_used_bytes{application=\"$application\", instance=\"$instance\", area=\"nonheap\"})*100/sum(jvm_memory_max_bytes{application=\"$application\",instance=\"$instance\", area=\"nonheap\"})",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "",
          "refId": "A",
          "step": 14400,
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Non-Heap Used",
      "type": "gauge"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 1
      },
      "hiddenSeries": false,
      "id": 66,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "process_files_open{application=\"$application\", instance=\"$instance\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Open Files",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "process_files_max{application=\"$application\", instance=\"$instance\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Max Files",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "Process Open Files",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "locale",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "dateTimeAsIso"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 6,
        "x": 0,
        "y": 4
      },
      "id": 56,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "process_start_time_seconds{application=\"$application\", instance=\"$instance\"}*1000",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "",
          "metric": "",
          "refId": "A",
          "step": 14400,
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Start time",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 7
      },
      "hiddenSeries": false,
      "id": 95,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "system_cpu_usage{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "System CPU Usage",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "process_cpu_usage{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Process CPU Usage",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "CPU Usage",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 7
      },
      "hiddenSeries": false,
      "id": 96,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "system_load_average_1m{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Load Average [1m]",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "system_cpu_count{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "CPU Core Size",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "Load Average",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 14
      },
      "id": 48,
      "panels": [],
      "title": "JVM Statistics - Memory",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 0,
        "y": 15
      },
      "hiddenSeries": false,
      "id": 85,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": "memory_pool_heap",
      "repeatDirection": "h",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_memory_used_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_heap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Used",
          "refId": "C",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_memory_committed_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_heap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Commited",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_memory_max_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_heap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Max",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "$memory_pool_heap (heap)",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 0,
        "y": 23
      },
      "hiddenSeries": false,
      "id": 88,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "repeat": "memory_pool_nonheap",
      "repeatDirection": "h",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_memory_used_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_nonheap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Used",
          "refId": "C",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_memory_committed_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_nonheap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Commited",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_memory_max_bytes{instance=\"$instance\", application=\"$application\", id=\"$memory_pool_nonheap\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Max",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "$memory_pool_nonheap (non-heap)",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 23
      },
      "hiddenSeries": false,
      "id": 80,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(jvm_classes_unloaded_total{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Classes Unloaded",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "Classes Unloaded",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "decimals": 0,
      "fill": 1,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 39
      },
      "id": 50,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_classes_loaded{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Classes Loaded",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Classes Loaded",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": 0,
          "format": "locale",
          "label": "",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 39
      },
      "hiddenSeries": false,
      "id": 83,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "8.5.2",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_buffer_memory_used_bytes{instance=\"$instance\", application=\"$application\", id=\"mapped\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Used Bytes",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_buffer_total_capacity_bytes{instance=\"$instance\", application=\"$application\", id=\"mapped\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Capacity Bytes",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "timeRegions": [],
      "title": "Mapped Buffers",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 46
      },
      "id": 78,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(jvm_gc_memory_allocated_bytes_total{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "allocated",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "irate(jvm_gc_memory_promoted_bytes_total{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "promoted",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Memory Allocate/Promote",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 47
      },
      "id": 82,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_buffer_memory_used_bytes{instance=\"$instance\", application=\"$application\", id=\"direct\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Used Bytes",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_buffer_total_capacity_bytes{instance=\"$instance\", application=\"$application\", id=\"direct\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Capacity Bytes",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Direct Buffers",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 54
      },
      "id": 68,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "jvm_threads_daemon{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Daemon",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_threads_live{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Live",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "jvm_threads_peak{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Peak",
          "refId": "C",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Threads",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 62
      },
      "id": 72,
      "panels": [],
      "title": "JVM Statistics - GC",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 10,
        "w": 12,
        "x": 0,
        "y": 63
      },
      "id": 74,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideEmpty": true,
        "hideZero": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(jvm_gc_pause_seconds_count{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{action}} [{{cause}}]",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "GC Count",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "locale",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 10,
        "w": 12,
        "x": 12,
        "y": 63
      },
      "id": 76,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "hideEmpty": true,
        "hideZero": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(jvm_gc_pause_seconds_sum{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{action}} [{{cause}}]",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "GC Stop the World Duration",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 73
      },
      "id": 34,
      "panels": [],
      "title": "HikariCP Statistics",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 74
      },
      "id": 44,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "none",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "hikaricp_connections{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Connections Size",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 8,
        "w": 20,
        "x": 4,
        "y": 74
      },
      "id": 36,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "hideEmpty": true,
        "hideZero": false,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": true,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hikaricp_connections_active{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Active",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "hikaricp_connections_idle{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Idle",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "hikaricp_connections_pending{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Pending",
          "refId": "C",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Connections",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "none"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 78
      },
      "id": 46,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "none",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "hikaricp_connections_timeout_total{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Connection Timeout Count",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 82
      },
      "id": 38,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hikaricp_connections_creation_seconds_sum{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"} / hikaricp_connections_creation_seconds_count{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Creation Time",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Connection Creation Time",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 8,
        "y": 82
      },
      "id": 42,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hikaricp_connections_usage_seconds_sum{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"} / hikaricp_connections_usage_seconds_count{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Usage Time",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Connection Usage Time",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 82
      },
      "id": 40,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "hikaricp_connections_acquire_seconds_sum{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"} / hikaricp_connections_acquire_seconds_count{instance=\"$instance\", application=\"$application\", pool=\"$hikaricp\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Acquire Time",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Connection Acquire Time",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 88
      },
      "id": 18,
      "panels": [],
      "title": "HTTP Statistics",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 89
      },
      "id": 4,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(http_server_requests_seconds_count{instance=\"$instance\", application=\"$application\", uri!~\".*actuator.*\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{method}} [{{status}}] - {{uri}}",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Request Count",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 96
      },
      "id": 2,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": true,
        "min": true,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(http_server_requests_seconds_sum{instance=\"$instance\", application=\"$application\", exception=\"None\", uri!~\".*actuator.*\"}[5m]) / irate(http_server_requests_seconds_count{instance=\"$instance\", application=\"$application\", exception=\"None\", uri!~\".*actuator.*\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{method}} [{{status}}] - {{uri}}",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Response Time",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "s",
          "label": "",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 103
      },
      "id": 22,
      "panels": [],
      "title": "Tomcat Statistics",
      "type": "row"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "locale"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 104
      },
      "id": 28,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "none",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "tomcat_global_error_total{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Total Error Count",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "decimals": 0,
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 9,
        "x": 4,
        "y": 104
      },
      "id": 24,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "tomcat_sessions_active_current{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "active sessions",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Active Sessions",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "label": "",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 11,
        "x": 13,
        "y": 104
      },
      "id": 26,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "irate(tomcat_global_sent_bytes_total{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Sent Bytes",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "irate(tomcat_global_received_bytes_total{instance=\"$instance\", application=\"$application\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Recieved Bytes",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Sent & Recieved Bytes",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "match": "null",
                "result": {
                  "text": "N/A"
                }
              },
              "type": "special"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "locale"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 3,
        "w": 4,
        "x": 0,
        "y": 108
      },
      "id": 32,
      "links": [],
      "maxDataPoints": 100,
      "options": {
        "colorMode": "none",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "8.5.2",
      "targets": [
        {
          "expr": "tomcat_threads_config_max{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "title": "Thread Config Max",
      "type": "stat"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 13,
        "x": 0,
        "y": 111
      },
      "id": 30,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "tomcat_threads_current{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Current thread",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        },
        {
          "expr": "tomcat_threads_busy{instance=\"$instance\", application=\"$application\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "Current thread busy",
          "refId": "B",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "Threads",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "collapsed": false,
      "datasource": {
        "type": "prometheus",
        "uid": "DxTyMDjnk"
      },
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 118
      },
      "id": 8,
      "panels": [],
      "title": "Logback Statistics",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 0,
        "y": 119
      },
      "id": 6,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "",
          "expr": "irate(logback_events_total{instance=\"$instance\", application=\"$application\", level=\"info\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "info",
          "rawSql": "SELECT\n  $__time(time_column),\n  value1\nFROM\n  metric_table\nWHERE\n  $__timeFilter(time_column)\n",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "INFO logs",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 12,
        "x": 12,
        "y": 119
      },
      "id": 10,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "",
          "expr": "irate(logback_events_total{instance=\"$instance\", application=\"$application\", level=\"error\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "error",
          "rawSql": "SELECT\n  $__time(time_column),\n  value1\nFROM\n  metric_table\nWHERE\n  $__timeFilter(time_column)\n",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "ERROR logs",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 8,
        "x": 0,
        "y": 126
      },
      "id": 14,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "",
          "expr": "irate(logback_events_total{instance=\"$instance\", application=\"$application\", level=\"warn\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "warn",
          "rawSql": "SELECT\n  $__time(time_column),\n  value1\nFROM\n  metric_table\nWHERE\n  $__timeFilter(time_column)\n",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "WARN logs",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 8,
        "x": 8,
        "y": 126
      },
      "id": 16,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "",
          "expr": "irate(logback_events_total{instance=\"$instance\", application=\"$application\", level=\"debug\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "debug",
          "rawSql": "SELECT\n  $__time(time_column),\n  value1\nFROM\n  metric_table\nWHERE\n  $__timeFilter(time_column)\n",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "DEBUG logs",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": {
        "type": "prometheus",
        "uid": "${DS_PROMETHEUS}"
      },
      "fill": 1,
      "gridPos": {
        "h": 7,
        "w": 8,
        "x": 16,
        "y": 126
      },
      "id": 20,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": true,
        "max": true,
        "min": true,
        "show": true,
        "total": true,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "",
          "expr": "irate(logback_events_total{instance=\"$instance\", application=\"$application\", level=\"trace\"}[5m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "trace",
          "rawSql": "SELECT\n  $__time(time_column),\n  value1\nFROM\n  metric_table\nWHERE\n  $__timeFilter(time_column)\n",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "thresholds": [],
      "title": "TRACE logs",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "mode": "time",
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "none",
          "logBase": 1,
          "show": true
        },
        {
          "format": "short",
          "logBase": 1,
          "show": true
        }
      ],
      "yaxis": {
        "align": false
      }
    }
  ],
  "refresh": "5s",
  "schemaVersion": 36,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": [
      {
        "current": {},
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "definition": "label_values(jvm_classes_loaded_classes, instance)",
        "hide": 0,
        "includeAll": false,
        "label": "Instance",
        "multi": false,
        "name": "instance",
        "options": [],
        "query": {
          "query": "label_values(jvm_classes_loaded_classes, instance)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "current": {},
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "definition": "label_values(jvm_classes_loaded_classes{instance=\"$instance\"}, application)",
        "hide": 0,
        "includeAll": false,
        "label": "Application",
        "multi": false,
        "name": "application",
        "options": [],
        "query": {
          "query": "label_values(jvm_classes_loaded_classes{instance=\"$instance\"}, application)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "current": {},
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "definition": "label_values(hikaricp_connections{instance=\"$instance\", application=\"$application\"}, pool)",
        "hide": 0,
        "includeAll": false,
        "label": "HikariCP-Pool",
        "multi": false,
        "name": "hikaricp",
        "options": [],
        "query": {
          "query": "label_values(hikaricp_connections{instance=\"$instance\", application=\"$application\"}, pool)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "current": {},
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "definition": "",
        "hide": 0,
        "includeAll": true,
        "label": "Memory Pool (heap)",
        "multi": false,
        "name": "memory_pool_heap",
        "options": [],
        "query": {
          "query": "label_values(jvm_memory_used_bytes{application=\"$application\", instance=\"$instance\", area=\"heap\"},id)",
          "refId": "Prometheus-memory_pool_heap-Variable-Query"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "current": {},
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
        "definition": "",
        "hide": 0,
        "includeAll": true,
        "label": "Memory Pool (nonheap)",
        "multi": false,
        "name": "memory_pool_nonheap",
        "options": [],
        "query": {
          "query": "label_values(jvm_memory_used_bytes{application=\"$application\", instance=\"$instance\", area=\"nonheap\"},id)",
          "refId": "Prometheus-memory_pool_nonheap-Variable-Query"
        },
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 1,
        "tagValuesQuery": "",
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-5m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "time_options": [
      "5m",
      "15m",
      "1h",
      "6h",
      "12h",
      "24h",
      "2d",
      "7d",
      "30d"
    ]
  },
  "timezone": "",
  "title": "Spring Boot Statistics",
  "uid": "sOae4vCnk",
  "version": 3,
  "weekStart": ""
}
Requires a lot of configuration stuff to do so
here i am using some predefined json data configured
click on + ---> import ---> paste the json that we have
click on load
select Promotheus dropdown ---> select Prometheus Micro services(we named it already)
click on import
we can see the dashboard for all these micro services

DockerFile
----------
# Use the OpenJDK 17 base image
FROM openjdk:17   -->The base image is an official OpenJDK image with Java 17 installed. if not pulls it from docker hub

# Expose port 3004
EXPOSE 3004   --> service is exposed at port no:3004

# Copy the entire project directory into the container
ADD target/driftin2-interviewing-service-0.0.1-SNAPSHOT.jar driftin2-interviewing-service-0.0.1-SNAPSHOT.jar --->copy local folder path to container path
ADD src/main/resources/ src/main/resources/  --->copy local folder path to container path
ADD target/ target/ --->copy local folder path to container path

# Set the entrypoint command with parameters as variables
ARG PROFILE                 --> this is the argument supplied at the time of building the image
ENV PROFILE=${PROFILE}      --> Sets an environment variable PROFILE in the container using the value of the PROFILE build-time argument.
ARG PROPERTIES_FILE         --> this is the argument supplied at the time of building the image
ENV PROPERTIES_FILE=${PROPERTIES_FILE} --> Sets an environment variable PROPERTIES_FILE in the container using the value of the PROPERTIES_FILE at build-time
ARG AWS_ACCESS_KEY_ID  --> this is the argument supplied at the time of building the image
ENV AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID --> Sets an environment variable AWS_ACCESS_KEY_ID in the container using the value of the AWS_ACCESS_KEY_ID at build.
ARG AWS_SECRET_ACCESS_KEY  --> this is the argument supplied at the time of building the image
ENV AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY --> Sets an environment variable AWS_SECRET_ACCESS_KEY in the container using the value of the AWS_SECRET_ACCESS_KEY at build-time.

# Verify if the properties file is copied correctly
RUN ls -l /src/main/resources/

ENTRYPOINT ["java", "-jar", "/driftin2-interviewing-service-0.0.1-SNAPSHOT.jar", "--spring.profiles.active=${PROFILE}", "--spring.config.location=${PROPERTIES_FILE}"]  --> Specifies the command to run when the container starts. It launches the Spring Boot application with the specified profile and properties file

docker build --build-arg PROFILE=dev --build-arg PROPERTIES_FILE=/path/to/application.properties --build-arg AWS_ACCESS_KEY_ID=my-access-key --build-arg AWS_SECRET_ACCESS_KEY=my-secret-key -t driftin2-interviewing-service-dev .

docker setup for project
=========================
docker network create microservices-net
docker run --network microservices-net -p 5432:5432 --name ms_pg_sql -e POSTGRES_USER=nagababu -e POSTGRES_PASSWORD=nagababu -d postgres 
docker run --network microservices-net -p 80:80 -e PGADMIN_DEFAULT_EMAIL=nagababu.ba@gmail.com -e PGADMIN_DEFAULT_PASSWORD=nagababu --name ms_pgadmin -d dpage/pgadmin4
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
logstash.bat -f C:\logstash\config\logstash.conf
https://www.fosstechnix.com/how-to-install-elastic-stack-on-ubuntu-24-04/
docker run --network microservices-net -p 9411:9411 openzipkin/zipkin:latest
docker run -p 8080:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:26.0.7 start-dev
docker run --name ms-mail-dev -d  -p 1025:1025  -p 1080:1080 maildev/maildev

Mail Template
=============
Docker setup for configuration

  mail-dev:
    container_name: ms-mail-dev
    image: maildev/maildev
    ports:
      - 1080:1080
      - 1025:1025
 
dependencies
------------
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

Preaparing Mail Messaging Environment
-------------------------------------
MimeMessage mimeMessage = mailSender.createMimeMessage();
MimeMessageHelper messageHelper = new MimeMessageHelper(mimeMessage, MimeMessageHelper.MULTIPART_MODE_MIXED_RELATED, UTF_8.name());

messageHelper.setFrom("awsnagababu@gmail.com");--->from address
messageHelper.setSubject("SOME SUBJECT"); --->subject
 Map<String, Object> variables = new HashMap<>();
 variables.put("customerName", customerName);
 variables.put("amount", amount);
 variables.put("orderReference", orderReference);
 Context context = new Context();
 context.setVariables(variables);
Dynamically renders data by substituing the context information

payment-confirmation.html
--------------------------
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Confirmation</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
        }
        .container {
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }
        h1 {
            color: #333;
        }
        p {
            color: #555;
        }
        .button {
            display: inline-block;
            padding: 10px 20px;
            text-align: center;
            text-decoration: none;
            color: #fff;
            background-color: #007BFF;
            border-radius: 5px;
        }
        .footer {
            margin-top: 20px;
            padding-top: 10px;
            border-top: 1px solid #ddd;
            text-align: center;
        }
    </style>
</head>
<body>
<div class="container">
    <h1>Payment Confirmation</h1>
    <p>Dear <span th:text="${customerName}"></span>,</p>
    <p>Your payment of $<span th:text="${amount}"></span> has been successfully processed.</p>
    <p>Order reference: <span th:text="${orderReference}"></span></p>
    <p>Thank you for choosing our service. If you have any questions, feel free to contact us.</p>
    <div class="footer">
        <p>This is an automated message. Please do not reply to this email.</p>
        <p>&copy; 2024 AlibouCoding. All rights reserved.</p>
    </div>
</div>
</body>
</html>

String htmlTemplate = templateEngine.process("payment-confirmation.html", context);
messageHelper.setText(htmlTemplate, true);
messageHelper.setTo("nagababu.ba@gmail.com");
mailSender.send(mimeMessage);

Webclient for service-service communication
===========================================
dependency
----------
webflux

bean definition
---------------
@Bean
    public WebClient webClient(WebClient.Builder builder) {
    return builder
	.baseUrl("https://example.com") // Default base URL (optional)
	.defaultHeader("Content-Type", "application/json") // Default headers (optional)
	.build();
    }

 customizing the bean
 -----------------
 WebClient customWebClient = webClient.mutate()
        .baseUrl("https://another-api.com")
        .defaultHeader("Authorization", "Bearer token")
        .build();

sending get request
--------------------
String someContent = webClient.get()
	.uri("/api/resource")
	.retrieve()
	.bodyToMono(String.class)
	.block(); 

Sending Post Request
--------------------
MyResponse response = webClient.post()
	.uri(url) // Specify the endpoint URI
	.bodyValue(requestBody) // Set the request body
	.retrieve() // Trigger the request and retrieve the response
	.bodyToMono(MyResponse.class) // Map response body to a Mono of MyResponse
	.block(); // Blocking to get the response (avoid blocking in reactive systems)

Sending Put Request
-------------------
MyResponse response = webClient.put()
	.uri(url) // Specify the endpoint URI
	.bodyValue(requestBody) // Set the request body
	.retrieve() // Trigger the request and retrieve the response
	.bodyToMono(MyResponse.class) // Map response body to a Mono of MyResponse
	.block(); // Blocking to get the response (avoid blocking in reactive systems)

 Sending Delete Request
 ----------------------
 webClient.delete()
            .uri(url)
            .retrieve()
            .bodyToMono(Void.class)
            .subscribe(response -> System.out.println("Async: Resource deleted successfully"));

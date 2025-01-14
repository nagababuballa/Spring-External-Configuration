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
Requires a lot of configuration stuff to do so
here i am using some predefined json data configured
click on + ---> import ---> paste the json that we have on grafafana_dashboard.json
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

Validation
==========

Dependency
----------
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
@Valid ---> used at controller level for validating request body

public class User {
    @NotNull(message = "Username cannot be null")
    @Size(min = 2, max = 50, message = "Username should be between 2 and 50 characters")
    private String username;
    @NotNull(message = "Email cannot be null")
    @Email(message = "Email should be valid")
    private String email;
    @Min(value = 18, message = "Age should be at least 18")
    private int age;
    @NotNull(message = "Date of Birth cannot be null")
    @Past(message = "Date of birth must be in the past")
    private LocalDate dateOfBirth;
    // Getters and setters
}

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

Configuring Open API Documentation SWAGGER
===========================================
Dependency
----------
<dependency>
   <groupId>org.springdoc</groupId>
   <artifactId>springdoc-openapi-ui</artifactId>
   <version>2.1.0</version> <!-- Use the latest version -->
</dependency>

Customizing the API Bean
------------------------

@Bean
public OpenAPI customOpenAPI() {
 return new OpenAPI()
        .info(new Info()
        .title("Microservices API Documentation")
        .version("1.0")
        .description("This is the API documentation for our microservices system"));
    }

 Properties
 ----------

 # Customize the Swagger UI path
springdoc.swagger-ui.path=/api-docs

# Customize the OpenAPI spec path
springdoc.api-docs.path=/v3/api-docs

Controller Level
----------------
@Operation(summary = "Authentication service")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Authentication service",
                    content = {@Content(mediaType = "application/json",
                            schema = @Schema(implementation = SwaggerAuthResponseVO.class))}),
            @ApiResponse(responseCode = "401", description = "Invalid credentials/token supplied",
                    content = @Content)
    })   

Generating Documentation using Java Doc Tool
--------------------------------------------

javadoc -d /path/to/output -sourcepath /path/to/src -subpackages com.example ---> this statement generates the documenation for all the classes which are present in and sub package of com.example

javadoc -d /path/to/output /path/to/YourClass.java ---> java documentation for a single class

Testing
=======

add the Jacoco plugin to measure the code coverage

<plugin>
   <groupId>org.jacoco</groupId>
   <artifactId>jacoco-maven-plugin</artifactId>
   <version>0.8.12</version>
   <executions>
	<execution>
		<goals>
		   <goal>prepare-agent</goal>
		</goals>
	</execution>
	<execution>
		<id>report</id>
		<phase>prepare-package</phase>
		<goals>
		   <goal>report</goal>
		</goals>
	</execution>
   </executions>
</plugin>

prepare-agent: Sets up the agent to collect coverage data during tests.
report: Generates the coverage report after the tests are executed.
the report will be placed in the target/site/jacoco/ directory by default.
open index.html

Maven Surefire Plugin
---------------------

If the plugin is not explicitly declared, Maven will use its internal default version of the Surefire Plugin to run the tests.

generate the default test reports (target/surefire-reports/)

Development side
----------------
1)JUnit is used for the overall test structure and lifecycle (e.g., @Test annotation).
2)Mockito is used for mocking the MyService dependency.
3)okhttp: Used for making HTTP requests, but it's not focused on unit testing. You might use it in integration tests where HTTP interactions are required.
4)mockwebserver: Used for mocking HTTP server responses, useful in integration tests or in unit tests that involve HTTP communication.
5)assertj-core: Provides expressive assertions, useful for writing unit test assertions that are more readable and powerful.

Under the hood Junit and mockito by default included with spring-bbot-starter-test

Dependencies
------------

<dependency>
   <groupId>com.squareup.okhttp3</groupId>
   <artifactId>okhttp</artifactId>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>com.squareup.okhttp3</groupId>
   <artifactId>mockwebserver</artifactId>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>org.assertj</groupId>
   <artifactId>assertj-core</artifactId>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>test</artifactId>
</dependency>

controller
---------
Use @WebMvcTest with MockMvc for controller-only tests.

@WebMvcTest(MyController.class)
public class MyControllerTest {
    @Autowired
    private MockMvc mockMvc;
    @MockBean
    private MyService myService;
    @BeforeEach
    public void setup() {
        MockitoAnnotations.openMocks(this);
    }
    @Test
    public void testGetData() throws Exception {
        when(myService.getData()).thenReturn("Mocked Data");
        mockMvc.perform(get("/api/data"))
               .andExpect(status().isOk())
               .andExpect(content().string("Mocked Data"));
    }
    @Test
    public void testPostData() throws Exception {
        String requestBody = "{\"name\":\"test\"}";
        mockMvc.perform(post("/api/data")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.message").value("Data created"));
    }
    @Test
    public void testPutData() throws Exception {
        String requestBody = "{\"name\":\"updated\"}";
        mockMvc.perform(put("/api/data/1")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.message").value("Data updated"));
    }
    @Test
    public void testDeleteData() throws Exception {
        mockMvc.perform(delete("/api/data/1"))
               .andExpect(status().isNoContent());
    }
}

Service
-------
@RunWith(SpringRunner.class)
@SpringBootTest  // Spring Boot test with full context (can be used for service testing)
public class MyServiceTest {
    @Autowired
    private MyService myService;  // Service under test
    @MockBean
    private MyRepository myRepository;  // Mock repository dependency
    @Test
    public void testGetData() {
        // Arrange: Mock the repository's behavior
        when(myRepository.findDataById(1)).thenReturn(new Data("Sample Data"));
        // Act: Call the service method
        Data result = myService.getDataById(1);
        // Assert: Verify the result
        assertNotNull(result);
        assertEquals("Sample Data", result.getData());
    }
    @Test
    public void testCreateData() {
        // Arrange: Create the data to be saved
        Data newData = new Data("New Data");   
        // Act: Call the service method
        when(myRepository.save(any(Data.class))).thenReturn(newData);
        Data savedData = myService.createData(newData);
        // Assert: Verify that the data was saved correctly
        assertNotNull(savedData);
        assertEquals("New Data", savedData.getData());
    }
}

Repository Only Test
--------------------
@RunWith(SpringRunner.class)
@DataJpaTest  // Automatically sets up an in-memory database for testing JPA
public class MyRepositoryTest {
    @Autowired
    private MyRepository myRepository;  // Injecting the repository under test
    @Test
    public void testFindById() {
        // Arrange: Save an entity into the in-memory database
        MyEntity savedEntity = myRepository.save(new MyEntity("Test Data"));
        // Act: Fetch the entity using the repository
        Optional<MyEntity> foundEntity = myRepository.findById(savedEntity.getId());
        // Assert: Verify that the entity is correctly fetched
        assertTrue(foundEntity.isPresent());
        assertEquals("Test Data", foundEntity.get().getName());
    }
    @Test
    public void testFindByName() {
        // Arrange: Save an entity
        myRepository.save(new MyEntity("John Doe"));
        // Act: Fetch the entity by name
        List<MyEntity> entities = myRepository.findByName("John Doe");
        // Assert: Verify the list contains the entity
        assertEquals(1, entities.size());
        assertEquals("John Doe", entities.get(0).getName());
    }
    @Test
    public void testDeleteById() {
        // Arrange: Save an entity
        MyEntity savedEntity = myRepository.save(new MyEntity("Test Data"));
        // Act: Delete the entity by ID
        myRepository.deleteById(savedEntity.getId());
        // Assert: Verify that the entity is deleted
        Optional<MyEntity> deletedEntity = myRepository.findById(savedEntity.getId());
        assertFalse(deletedEntity.isPresent());
    }
}

Integration Test
----------------
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
public class UserServiceIntegrationTest {
    private MockWebServer mockWebServer;
    @Autowired
    private UserService userService;
    @Before
    public void setUp() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
        // Override WebClient base URL for testing
        ReflectionTestUtils.setField(userService, "webClient",
                WebClient.builder()
                        .baseUrl(mockWebServer.url("/").toString())
                        .build());
    }
    @After
    public void tearDown() throws IOException {
        mockWebServer.shutdown();
    }
    @Test
    public void testGetUserById() throws Exception {
        // Arrange: Mock the external API response
        String mockResponse = """
                {
                    "id": "1",
                    "name": "John Doe",
                    "email": "john.doe@example.com"
                }
                """;
        mockWebServer.enqueue(new MockResponse()
                .setBody(mockResponse)
                .addHeader("Content-Type", "application/json"));
        // Act: Call the service method
        Mono<User> userMono = userService.getUserById("1");
        User user = userMono.block();
        // Assert: Verify the response
        assertNotNull(user);
        assertEquals("1", user.getId());
        assertEquals("John Doe", user.getName());
        assertEquals("john.doe@example.com", user.getEmail());
        // Verify the request made to MockWebServer
        RecordedRequest recordedRequest = mockWebServer.takeRequest();
        assertEquals("GET", recordedRequest.getMethod());
        assertEquals("/users/1", recordedRequest.getPath());
    }
}

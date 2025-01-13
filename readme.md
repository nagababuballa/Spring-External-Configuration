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

postman testing
================

Method: Post 
Uri    : http://localhost:3102/api/v1/customers
RequestBody :
  {
   "firstname":"Balla",
   "lastname":"Nagababu",
   "email":"nagababu.ba@gmail.com",
   "address":{
    "street":"Pitapuram Colony",
    "houseNumber":"7-112/3",
    "zipCode":"530003"
   }

Upon success new customer record will be created

MimeMessage mimeMessage = mailSender.createMimeMessage();
MimeMessageHelper messageHelper = new MimeMessageHelper(mimeMessage, MimeMessageHelper.MULTIPART_MODE_MIXED_RELATED, UTF_8.name());

messageHelper.setFrom("contact@aliboucoding.com");
messageHelper.setSubject(PAYMENT_CONFIRMATION.getSubject());

 Map<String, Object> variables = new HashMap<>();
 variables.put("customerName", customerName);
 variables.put("amount", amount);
 variables.put("orderReference", orderReference);
 Context context = new Context();
 context.setVariables(variables);
String htmlTemplate = templateEngine.process(templateName, context);
messageHelper.setText(htmlTemplate, true);
messageHelper.setTo(destinationEmail);
mailSender.send(mimeMessage);

=====================================================================================
ResponseEntity<HackerRankCandidateInfo> candidateResultResponseEntity = this.webClientForHR
                    .post()
                    .uri(testId + HR_CANDIDATES)
                    .body(BodyInserters.fromValue(inviteRequest))
                    .header(HttpHeaders.AUTHORIZATION, hackerrankAPIToken)
                    .retrieve()
                    .toEntity(new ParameterizedTypeReference<HackerRankCandidateInfo>() {
                    })
                    .block();
=========================================================================================
private MockWebServer mockBackEnd;

 
    
 @BeforeEach
    void setUp() throws IOException {
        mockBackEnd = new MockWebServer();
        mockBackEnd.start();
        String baseUrl = mockBackEnd.url("/").toString();
        webClient = WebClient.create(baseUrl);
        hackerRankAPI = new HackerRankAPI(webClient);
    }

 @AfterEach
    void tearDown() throws IOException {
        mockBackEnd.shutdown();
    }

@Test
    void testGetHackerRankTestInvite() {
        mockBackEnd.enqueue(new MockResponse()
                .setResponseCode(HttpStatus.OK.value())
                .setHeader("Content-Type", "application/json")
                .setBody(responseBody));
        ResponseEntity<HackerRankCandidateInviteResponse> responseEntity =
                hackerRankAPI.getHackerRankTestInvite(new HackerRankCandidateInviteRequest(), "1234");
        HackerRankCandidateInviteResponse hackerRankCandidateInviteResponse = responseEntity.getBody();
        HackerRankCandidateInviteResponse expectedResponse = getHackerRankCandidateInviteResponse();
        assertEquals(hackerRankCandidateInviteResponse.getHackerRankCandidateId(), expectedResponse.getHackerRankCandidateId());
        assertEquals(hackerRankCandidateInviteResponse.getTestLink(), expectedResponse.getTestLink());
        assertEquals(hackerRankCandidateInviteResponse.getEmail(), expectedResponse.getEmail());
    }


@Test
void testGetHackerRankTestInvite() {
    String responseBody = "{\n" +
            "  \"test_link\": \"test link url\",\n" +
            "  \"email\": \"email\",\n" +
            "  \"id\": \"1234\"\n" +
            "}";
    mockBackEnd.enqueue(new MockResponse()
            .setResponseCode(HttpStatus.OK.value())
            .setHeader("Content-Type", "application/json")
            .setBody(responseBody));
    ResponseEntity<HackerRankCandidateInviteResponse> responseEntity =
            hackerRankAPI.getHackerRankTestInvite(new HackerRankCandidateInviteRequest(), "1234");
    HackerRankCandidateInviteResponse hackerRankCandidateInviteResponse = responseEntity.getBody();
    HackerRankCandidateInviteResponse expectedResponse = getHackerRankCandidateInviteResponse();
    assertEquals(hackerRankCandidateInviteResponse.getHackerRankCandidateId(), expectedResponse.getHackerRankCandidateId());
    assertEquals(hackerRankCandidateInviteResponse.getTestLink(), expectedResponse.getTestLink());
    assertEquals(hackerRankCandidateInviteResponse.getEmail(), expectedResponse.getEmail());
}

private HackerRankCandidateInviteResponse getHackerRankCandidateInviteResponse() {
    HackerRankCandidateInviteResponse hackerRankCandidateInviteResponse = new HackerRankCandidateInviteResponse();
    hackerRankCandidateInviteResponse.setHackerRankCandidateId("1234");
    hackerRankCandidateInviteResponse.setTestLink("test link url");
    hackerRankCandidateInviteResponse.setEmail("email");
    return hackerRankCandidateInviteResponse;
}

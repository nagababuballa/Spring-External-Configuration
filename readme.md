docker network create microservices-net
docker run --network microservices-net -p 5432:5432 --name ms_pg_sql -e POSTGRES_USER=nagababu -e POSTGRES_PASSWORD=nagababu -d postgres 
docker run --network microservices-net -p 80:80 -e PGADMIN_DEFAULT_EMAIL=nagababu.ba@gmail.com -e PGADMIN_DEFAULT_PASSWORD=nagababu --name ms_pgadmin -d dpage/pgadmin4
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
logstash.bat -f C:\logstash\config\logstash.conf
https://www.fosstechnix.com/how-to-install-elastic-stack-on-ubuntu-24-04/
docker run --network microservices-net -p 9411:9411 openzipkin/zipkin:latest
docker run -p 8080:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:26.0.7 start-dev
docker run --name ms-mail-dev -d  -p 1025:1025  -p 1080:1080 maildev/maildev

@Bean
    public NewTopic orderTopic() {
        return TopicBuilder
                .name("order-topic")
                .build();
    }

Message<PaymentNotificationRequest> message = MessageBuilder
            .withPayload(request)
            .setHeader(TOPIC, "payment-topic")
            .build();

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

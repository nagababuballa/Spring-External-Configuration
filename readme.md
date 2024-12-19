docker network create microservices-net
docker run --network microservices-net -p 5432:5432 --name ms_pg_sql -e POSTGRES_USER=nagababu -e POSTGRES_PASSWORD=nagababu -d postgres 
docker run --network microservices-net -p 80:80 -e PGADMIN_DEFAULT_EMAIL=nagababu.ba@gmail.com -e PGADMIN_DEFAULT_PASSWORD=nagababu --name ms_pgadmin -d dpage/pgadmin4
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
docker run -p 8080:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:26.0.7 start-dev

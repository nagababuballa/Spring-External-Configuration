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

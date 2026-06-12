# Tutorial: Mensageria com RabbitMQ, eventos e RPC no Spring Boot

## Objetivo da aula

Vamos construir uma prática de mensageria com Java 17, Spring Boot e RabbitMQ.

A primeira parte cria dois microsserviços:

* `orders-service`, que recebe pedidos por HTTP, salva no MongoDB e publica o evento `pedido.criado`;
* `notifications-service`, que escuta esse evento e registra no log que uma notificação seria enviada.

Depois disso, vamos olhar para o RPC com RabbitMQ usando um exemplo menor:

* `checkout-service`, que pergunta se existe estoque;
* `stock-service`, que responde essa consulta pela mensageria.

O ponto mais importante da aula é entender a diferença entre:

* publicar um evento e não esperar resposta;
* enviar uma pergunta e esperar resposta.

## Resultado final

Na parte principal, teremos:

```text
orders-service        -> http://localhost:8081
notifications-service -> consumidor RabbitMQ
RabbitMQ UI           -> http://localhost:15672
MongoDB               -> localhost:27017
```

O teste principal será criar um pedido:

```bash
curl -X POST http://localhost:8081/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Ana Souza","customerEmail":"ana@email.com","total":149.90}'
```

Por baixo dos panos, a aplicação fará isto:

```text
Cliente
   |
   v
POST /api/v1/orders
   |
   v
OrderController
   |
   v
OrderService salva no MongoDB
   |
   v
OrderEventPublisher publica pedido.criado
   |
   v
RabbitMQ entrega para notifications-service
```

Na segunda parte, veremos uma comparação com RPC:

```text
checkout-service pergunta -> RabbitMQ -> stock-service responde
```

Essa segunda parte existe para comparação. Ela não deve ser o primeiro padrão usado quando o sistema só precisa avisar que algo aconteceu.

## Contexto

Na aula de message brokers, já vimos a parte conceitual: producer, consumer, fila, exchange, routing key, binding, evento, pub/sub e RPC.

Agora a conversa muda para implementação.

Vamos pegar aquela ideia e transformar em serviços Spring Boot rodando de verdade:

* uma API de pedidos grava no MongoDB;
* a API publica o evento `pedido.criado` no RabbitMQ;
* outro serviço consome esse evento;
* no fim, comparamos com um exemplo pequeno de RPC usando RabbitMQ.

Use esta aula como prática guiada. Quando aparecer um termo de mensageria, ele será explicado só o suficiente para entender o arquivo que estamos criando.

## Explicação conceitual

Vamos usar o desenho abaixo como referência durante o tutorial:

```text
orders.exchange
   |
   | routing key pedido.criado
   v
notifications.order-created.queue
```

Na prática, isso aparece em três pontos do código:

* no `orders-service`, a classe `OrderEventPublisher` publica o evento usando `RabbitTemplate`;
* no `notifications-service`, a classe `OrderCreatedConsumer` escuta a fila usando `@RabbitListener`;
* nas classes `RabbitConfig`, configuramos exchange, queue, binding e conversão JSON.

O tutorial também cria um exemplo de RPC no final. Ele entra só como comparação com o pub/sub: no evento, publicamos e seguimos; no RPC, enviamos uma pergunta e esperamos resposta.

## Setup inicial

### Pré-requisitos

Você precisa ter:

* Java 17 instalado.
* Docker e Docker Compose.
* Um terminal.
* Um editor de código.

Use caminhos genéricos nos comandos:

```bash
cd /caminho/para/a/pasta-do-projeto
```

No Windows, use o terminal que preferir. O Gradle Wrapper aparece como `gradlew.bat`.

No Linux, use `./gradlew`.

### Criar a pasta da prática

```bash
mkdir rabbitmq-demo
cd rabbitmq-demo
```

### Criar o Docker Compose

Crie o arquivo `docker-compose.yml`:

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: orders-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: orders_user
      MONGO_INITDB_ROOT_PASSWORD: orders_pass
    ports:
      - "27017:27017"
    volumes:
      - orders_mongodb_data:/data/db

  rabbitmq:
    image: rabbitmq:3-management
    container_name: orders-rabbitmq
    restart: unless-stopped
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest

volumes:
  orders_mongodb_data:
```

Suba os containers:

```bash
docker compose up -d
```

O MongoDB será usado pelo `orders-service`.

O RabbitMQ será usado para publicar e consumir mensagens.

A interface web do RabbitMQ fica em:

```text
http://localhost:15672
```

Usuário e senha:

```text
guest
guest
```

## Passo a passo

### 1. Criar o orders-service

No Spring Initializr, crie um projeto com:

* Project: Gradle - Groovy.
* Language: Java.
* Spring Boot: versão estável 3.x. No momento da validação desta aula, usamos `3.5.14`.
* Group: `br.edu.upf`.
* Artifact: `orders-service`.
* Package name: `br.edu.upf.ordersservice`.
* Java: 17.
* Dependencies: Spring Web, Spring Data MongoDB, Validation, Lombok, Spring for RabbitMQ e Springdoc OpenAPI.

Depois de baixar e descompactar:

```bash
cd orders-service
```

Confira o `build.gradle`:

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.14'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'br.edu.upf'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-amqp'
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.14'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.amqp:spring-rabbit-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

O starter `spring-boot-starter-amqp` adiciona a integração do Spring com RabbitMQ.

O starter `spring-boot-starter-data-mongodb` permite criar repositories com Spring Data MongoDB.

### 2. Configurar o orders-service

Abra `src/main/resources/application.properties`:

```properties
spring.application.name=orders-service
server.port=${PORT:8081}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://orders_user:orders_pass@localhost:27017/orders_db?authSource=admin}

spring.rabbitmq.host=${RABBITMQ_HOST:localhost}
spring.rabbitmq.port=${RABBITMQ_PORT:5672}
spring.rabbitmq.username=${RABBITMQ_USERNAME:guest}
spring.rabbitmq.password=${RABBITMQ_PASSWORD:guest}

app.rabbitmq.order-exchange=orders.exchange
app.rabbitmq.order-created-routing-key=pedido.criado

springdoc.swagger-ui.path=/swagger-ui.html
```

As variáveis com `${...}` deixam a aplicação fácil de rodar localmente e em container.

Se nenhuma variável for informada, o Spring usa o valor depois dos dois-pontos.

### 3. Criar a estrutura de pacotes

Crie esta estrutura:

```text
src/main/java/br/edu/upf/ordersservice/
├── config/
├── controller/
├── dto/
├── mapper/
├── messaging/
├── model/
├── repository/
└── service/
```

Essa separação evita misturar HTTP, regra de negócio, banco de dados e mensageria na mesma classe.

### 4. Criar o model de pedido

Crie `src/main/java/br/edu/upf/ordersservice/model/Order.java`:

```java
package br.edu.upf.ordersservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.math.BigDecimal;
import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "orders")
public class Order {

    @Id
    private String id;

    private String customerName;
    private String customerEmail;
    private BigDecimal total;
    private Instant createdAt;
}
```

`@Document` diz que essa classe representa um documento no MongoDB.

`@Id` marca o campo usado como identificador.

`BigDecimal` é usado para dinheiro porque evita problemas de precisão comuns com `double`.

### 5. Criar os DTOs

Crie `src/main/java/br/edu/upf/ordersservice/dto/CreateOrderRequest.java`:

```java
package br.edu.upf.ordersservice.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;

import java.math.BigDecimal;

@Builder
public record CreateOrderRequest(
        @NotBlank String customerName,
        @NotBlank @Email String customerEmail,
        @NotNull @DecimalMin("0.01") BigDecimal total
) {
}
```

Crie `src/main/java/br/edu/upf/ordersservice/dto/OrderResponse.java`:

```java
package br.edu.upf.ordersservice.dto;

import lombok.Builder;

import java.math.BigDecimal;
import java.time.Instant;

@Builder
public record OrderResponse(
        String id,
        String customerName,
        String customerEmail,
        BigDecimal total,
        Instant createdAt
) {
}
```

Crie `src/main/java/br/edu/upf/ordersservice/dto/OrderCreatedEvent.java`:

```java
package br.edu.upf.ordersservice.dto;

import lombok.Builder;

import java.math.BigDecimal;
import java.time.Instant;

@Builder
public record OrderCreatedEvent(
        String orderId,
        String customerName,
        String customerEmail,
        BigDecimal total,
        Instant createdAt
) {
}
```

O request representa o que entra pela API.

O response representa o que sai.

O evento representa a mensagem enviada para o RabbitMQ.

Os três ficam separados porque cada um tem uma responsabilidade diferente.

### 6. Criar o mapper

Crie `src/main/java/br/edu/upf/ordersservice/mapper/OrderMapper.java`:

```java
package br.edu.upf.ordersservice.mapper;

import br.edu.upf.ordersservice.dto.OrderCreatedEvent;
import br.edu.upf.ordersservice.dto.OrderResponse;
import br.edu.upf.ordersservice.model.Order;

public final class OrderMapper {

    private OrderMapper() {
    }

    public static OrderResponse mapFrom(Order order) {
        if (order == null) {
            return null;
        }

        return OrderResponse.builder()
                .id(order.getId())
                .customerName(order.getCustomerName())
                .customerEmail(order.getCustomerEmail())
                .total(order.getTotal())
                .createdAt(order.getCreatedAt())
                .build();
    }

    public static OrderCreatedEvent toCreatedEvent(Order order) {
        if (order == null) {
            return null;
        }

        return OrderCreatedEvent.builder()
                .orderId(order.getId())
                .customerName(order.getCustomerName())
                .customerEmail(order.getCustomerEmail())
                .total(order.getTotal())
                .createdAt(order.getCreatedAt())
                .build();
    }
}
```

O mapper deixa explícita a conversão entre model, response e evento.

Essa conversão não fica no controller porque controller não deve montar resposta complexa nem conhecer detalhes do model.

### 7. Criar o repository

Crie `src/main/java/br/edu/upf/ordersservice/repository/OrderRepository.java`:

```java
package br.edu.upf.ordersservice.repository;

import br.edu.upf.ordersservice.model.Order;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface OrderRepository extends MongoRepository<Order, String> {
}
```

O Spring Data MongoDB cria a implementação por baixo dos panos.

Quando chamamos `save`, o Spring transforma isso em uma operação no MongoDB.

### 8. Configurar o RabbitMQ no producer

Crie `src/main/java/br/edu/upf/ordersservice/config/RabbitConfig.java`:

```java
package br.edu.upf.ordersservice.config;

import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.annotation.EnableRabbit;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@EnableRabbit
@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange ordersExchange(@Value("${app.rabbitmq.order-exchange}") String exchangeName) {
        return new TopicExchange(exchangeName, true, false);
    }

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

`TopicExchange` permite rotear mensagens usando routing keys.

O valor `true` deixa a exchange durável. Isso significa que ela continua existindo se o RabbitMQ reiniciar.

`Jackson2JsonMessageConverter` transforma o record Java em JSON antes de enviar a mensagem.

### 9. Criar o publisher

Crie `src/main/java/br/edu/upf/ordersservice/messaging/OrderEventPublisher.java`:

```java
package br.edu.upf.ordersservice.messaging;

import br.edu.upf.ordersservice.dto.OrderCreatedEvent;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class OrderEventPublisher {

    private static final Logger log = LoggerFactory.getLogger(OrderEventPublisher.class);

    private final RabbitTemplate rabbitTemplate;

    @Value("${app.rabbitmq.order-exchange}")
    private String orderExchange;

    @Value("${app.rabbitmq.order-created-routing-key}")
    private String orderCreatedRoutingKey;

    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(orderExchange, orderCreatedRoutingKey, event);
        log.info("Evento pedido.criado publicado para o pedido {}", event.orderId());
    }
}
```

`RabbitTemplate` é a classe do Spring usada para enviar mensagens ao RabbitMQ.

O service não precisa saber detalhes de exchange e routing key. Isso fica no publisher.

### 10. Criar o service

Crie `src/main/java/br/edu/upf/ordersservice/service/OrderService.java`:

```java
package br.edu.upf.ordersservice.service;

import br.edu.upf.ordersservice.dto.CreateOrderRequest;
import br.edu.upf.ordersservice.dto.OrderResponse;
import br.edu.upf.ordersservice.mapper.OrderMapper;
import br.edu.upf.ordersservice.messaging.OrderEventPublisher;
import br.edu.upf.ordersservice.model.Order;
import br.edu.upf.ordersservice.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderEventPublisher orderEventPublisher;

    public OrderResponse create(CreateOrderRequest request) {
        Order order = Order.builder()
                .customerName(request.customerName())
                .customerEmail(request.customerEmail())
                .total(request.total())
                .createdAt(Instant.now())
                .build();

        Order savedOrder = orderRepository.save(order);

        orderEventPublisher.publishOrderCreated(OrderMapper.toCreatedEvent(savedOrder));

        return OrderMapper.mapFrom(savedOrder);
    }

    public List<OrderResponse> findAll() {
        return orderRepository.findAll()
                .stream()
                .map(OrderMapper::mapFrom)
                .toList();
    }
}
```

Aqui está a regra principal da parte de eventos.

O service cria o model, salva no MongoDB, publica o evento e devolve o response.

O controller não participa dessa orquestração.

### 11. Criar o controller

Crie `src/main/java/br/edu/upf/ordersservice/controller/OrderController.java`:

```java
package br.edu.upf.ordersservice.controller;

import br.edu.upf.ordersservice.dto.CreateOrderRequest;
import br.edu.upf.ordersservice.dto.OrderResponse;
import br.edu.upf.ordersservice.service.OrderService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @GetMapping
    public List<OrderResponse> findAll() {
        return orderService.findAll();
    }
}
```

`@Valid` ativa as validações do request.

`@ResponseStatus(HttpStatus.CREATED)` faz o `POST` responder `201 Created`.

O controller só recebe HTTP e delega para o service.

### 12. Criar o notifications-service

Volte para a pasta principal:

```bash
cd /caminho/para/rabbitmq-demo
```

No Spring Initializr, crie outro projeto:

* Project: Gradle - Groovy.
* Language: Java.
* Spring Boot: versão estável 3.x. Use a mesma versão usada no `orders-service`, como `3.5.14`.
* Group: `br.edu.upf`.
* Artifact: `notifications-service`.
* Package name: `br.edu.upf.notificationsservice`.
* Java: 17.
* Dependencies: Spring for RabbitMQ e Lombok.

Abra o `build.gradle` do `notifications-service` e confira as dependências:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-amqp'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.amqp:spring-rabbit-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

O `spring-boot-starter-amqp` traz a integração com RabbitMQ.

O `jackson-databind` e o `jackson-datatype-jsr310` entram porque esse serviço não usa Spring Web. Como vamos converter JSON para um `record` que tem `Instant`, precisamos garantir que o Jackson saiba trabalhar com tipos de data e hora do Java.

Configure `src/main/resources/application.properties`:

```properties
spring.application.name=notifications-service
server.port=${PORT:8082}

spring.rabbitmq.host=${RABBITMQ_HOST:localhost}
spring.rabbitmq.port=${RABBITMQ_PORT:5672}
spring.rabbitmq.username=${RABBITMQ_USERNAME:guest}
spring.rabbitmq.password=${RABBITMQ_PASSWORD:guest}

app.rabbitmq.order-exchange=orders.exchange
app.rabbitmq.notification-queue=notifications.order-created.queue
app.rabbitmq.order-created-routing-key=pedido.criado
```

Esse serviço não precisa de controller para a prática principal.

Ele trabalha como consumer de mensagens.

### 13. Criar o DTO do evento no consumer

Crie `src/main/java/br/edu/upf/notificationsservice/dto/OrderCreatedEvent.java`:

```java
package br.edu.upf.notificationsservice.dto;

import lombok.Builder;

import java.math.BigDecimal;
import java.time.Instant;

@Builder
public record OrderCreatedEvent(
        String orderId,
        String customerName,
        String customerEmail,
        BigDecimal total,
        Instant createdAt
) {
}
```

O consumer precisa conhecer o formato da mensagem.

Em projetos maiores, esse contrato pode ir para uma biblioteca compartilhada ou para uma documentação de eventos. Para a aula, duplicar o record deixa mais simples enxergar que são dois serviços separados.

### 14. Configurar fila, exchange e binding

Crie `src/main/java/br/edu/upf/notificationsservice/config/RabbitConfig.java`:

```java
package br.edu.upf.notificationsservice.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.rabbit.annotation.EnableRabbit;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@EnableRabbit
@Configuration
public class RabbitConfig {

    @Bean
    public TopicExchange ordersExchange(@Value("${app.rabbitmq.order-exchange}") String exchangeName) {
        return new TopicExchange(exchangeName, true, false);
    }

    @Bean
    public Queue notificationQueue(@Value("${app.rabbitmq.notification-queue}") String queueName) {
        return new Queue(queueName, true);
    }

    @Bean
    public Binding orderCreatedBinding(
            Queue notificationQueue,
            TopicExchange ordersExchange,
            @Value("${app.rabbitmq.order-created-routing-key}") String routingKey
    ) {
        return BindingBuilder
                .bind(notificationQueue)
                .to(ordersExchange)
                .with(routingKey);
    }

    @Bean
    public MessageConverter messageConverter() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

A fila recebe somente mensagens que baterem com a routing key `pedido.criado`.

Essa ligação é feita pelo `Binding`.

O `JavaTimeModule` ensina o Jackson a converter tipos como `Instant`. Sem isso, o consumer pode receber a mensagem, mas falhar na conversão do JSON para `OrderCreatedEvent`.

### 15. Criar o consumer

Crie `src/main/java/br/edu/upf/notificationsservice/messaging/OrderCreatedConsumer.java`:

```java
package br.edu.upf.notificationsservice.messaging;

import br.edu.upf.notificationsservice.dto.OrderCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class OrderCreatedConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderCreatedConsumer.class);

    @RabbitListener(queues = "${app.rabbitmq.notification-queue}")
    public void consume(OrderCreatedEvent event) {
        log.info(
                "Pedido {} criado para {}. Enviar notificação para {}.",
                event.orderId(),
                event.customerName(),
                event.customerEmail()
        );
    }
}
```

`@RabbitListener` diz ao Spring para escutar uma fila.

Quando chegar uma mensagem, o Spring converte o JSON para `OrderCreatedEvent` e chama o método `consume`.

### 16. Rodar e testar os eventos

Com MongoDB e RabbitMQ rodando, abra dois terminais.

No primeiro:

```bash
cd /caminho/para/rabbitmq-demo/orders-service
./gradlew bootRun
```

No Windows:

```bash
gradlew.bat bootRun
```

No segundo:

```bash
cd /caminho/para/rabbitmq-demo/notifications-service
./gradlew bootRun
```

No Windows:

```bash
gradlew.bat bootRun
```

Crie um pedido:

```bash
curl -X POST http://localhost:8081/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Ana Souza","customerEmail":"ana@email.com","total":149.90}'
```

Liste os pedidos:

```bash
curl http://localhost:8081/api/v1/orders
```

No terminal do `notifications-service`, você deve ver um log parecido com:

```text
Pedido 665f... criado para Ana Souza. Enviar notificação para ana@email.com.
```

### 17. Comparar com RPC

Agora faz sentido olhar para o RPC, mas só como comparação.

No evento `pedido.criado`, o `orders-service` publica e segue a vida.

No RPC, o serviço que pergunta espera a resposta.

Para testar isso, crie dois projetos menores:

```text
checkout-service -> recebe HTTP na porta 8083
stock-service    -> responde mensagens pelo RabbitMQ
```

Crie o `checkout-service` pelo Spring Initializr com:

* Project: Gradle - Groovy.
* Language: Java.
* Spring Boot: mesma versão usada nos serviços anteriores, como `3.5.14`.
* Group: `br.edu.upf`.
* Artifact: `checkout-service`.
* Package name: `br.edu.upf.checkoutservice`.
* Java: 17.
* Dependencies: Spring Web, Validation, Spring for RabbitMQ e Lombok.

Crie o `stock-service` pelo Spring Initializr com:

* Project: Gradle - Groovy.
* Language: Java.
* Spring Boot: mesma versão usada nos serviços anteriores, como `3.5.14`.
* Group: `br.edu.upf`.
* Artifact: `stock-service`.
* Package name: `br.edu.upf.stockservice`.
* Java: 17.
* Dependencies: Spring for RabbitMQ e Lombok.

No `stock-service`, abra o `build.gradle` e garanta estas dependências:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-amqp'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.amqp:spring-rabbit-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

O `stock-service` também não usa Spring Web, então adicionamos Jackson explicitamente pelo mesmo motivo do `notifications-service`.

No `stock-service`, crie `src/main/java/br/edu/upf/stockservice/dto/StockCheckRequest.java`:

```java
package br.edu.upf.stockservice.dto;

import lombok.Builder;

@Builder
public record StockCheckRequest(
        String productId,
        Integer quantity
) {
}
```

Crie `src/main/java/br/edu/upf/stockservice/dto/StockCheckResponse.java`:

```java
package br.edu.upf.stockservice.dto;

import lombok.Builder;

@Builder
public record StockCheckResponse(
        String productId,
        Integer requestedQuantity,
        Integer availableQuantity,
        Boolean available
) {
}
```

Crie `src/main/java/br/edu/upf/stockservice/config/RabbitConfig.java`:

```java
package br.edu.upf.stockservice.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.annotation.EnableRabbit;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@EnableRabbit
@Configuration
public class RabbitConfig {

    @Bean
    public Queue stockRequestQueue(@Value("${app.rabbitmq.stock-request-queue}") String queueName) {
        return new Queue(queueName, true);
    }

    @Bean
    public MessageConverter messageConverter() {
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.registerModule(new JavaTimeModule());
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

Crie `src/main/java/br/edu/upf/stockservice/service/StockService.java`:

```java
package br.edu.upf.stockservice.service;

import br.edu.upf.stockservice.dto.StockCheckRequest;
import br.edu.upf.stockservice.dto.StockCheckResponse;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class StockService {

    private final Map<String, Integer> availableStock = Map.of(
            "PROD-100", 10,
            "PROD-200", 0,
            "PROD-300", 5
    );

    public StockCheckResponse check(StockCheckRequest request) {
        Integer availableQuantity = availableStock.getOrDefault(request.productId(), 0);

        return StockCheckResponse.builder()
                .productId(request.productId())
                .requestedQuantity(request.quantity())
                .availableQuantity(availableQuantity)
                .available(availableQuantity >= request.quantity())
                .build();
    }
}
```

Crie `src/main/java/br/edu/upf/stockservice/messaging/StockRequestConsumer.java`:

```java
package br.edu.upf.stockservice.messaging;

import br.edu.upf.stockservice.dto.StockCheckRequest;
import br.edu.upf.stockservice.dto.StockCheckResponse;
import br.edu.upf.stockservice.service.StockService;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class StockRequestConsumer {

    private static final Logger log = LoggerFactory.getLogger(StockRequestConsumer.class);

    private final StockService stockService;

    @RabbitListener(queues = "${app.rabbitmq.stock-request-queue}")
    public StockCheckResponse consume(StockCheckRequest request) {
        log.info("Recebida consulta de estoque para {}", request.productId());
        return stockService.check(request);
    }
}
```

No `checkout-service`, crie DTOs iguais para a mensagem e adicione validação no request HTTP:

```java
package br.edu.upf.checkoutservice.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;

@Builder
public record StockCheckRequest(
        @NotBlank String productId,
        @NotNull @Min(1) Integer quantity
) {
}
```

```java
package br.edu.upf.checkoutservice.dto;

import lombok.Builder;

@Builder
public record StockCheckResponse(
        String productId,
        Integer requestedQuantity,
        Integer availableQuantity,
        Boolean available
) {
}
```

Crie `src/main/java/br/edu/upf/checkoutservice/config/RabbitConfig.java`:

```java
package br.edu.upf.checkoutservice.config;

import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

Crie `src/main/java/br/edu/upf/checkoutservice/messaging/StockClient.java`:

```java
package br.edu.upf.checkoutservice.messaging;

import br.edu.upf.checkoutservice.dto.StockCheckRequest;
import br.edu.upf.checkoutservice.dto.StockCheckResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class StockClient {

    private final RabbitTemplate rabbitTemplate;

    @Value("${app.rabbitmq.stock-request-queue}")
    private String stockRequestQueue;

    public StockCheckResponse checkStock(StockCheckRequest request) {
        StockCheckResponse response = rabbitTemplate.convertSendAndReceiveAsType(
                stockRequestQueue,
                request,
                new ParameterizedTypeReference<>() {
                }
        );

        if (response == null) {
            throw new IllegalStateException("Serviço de estoque não respondeu");
        }

        return response;
    }
}
```

`convertSendAndReceiveAsType` envia a mensagem e bloqueia esperando uma resposta.

Esse é o ponto perigoso do RPC. Se o `stock-service` estiver lento, o `checkout-service` também fica lento.

Crie o service e o controller:

```java
package br.edu.upf.checkoutservice.service;

import br.edu.upf.checkoutservice.dto.StockCheckRequest;
import br.edu.upf.checkoutservice.dto.StockCheckResponse;
import br.edu.upf.checkoutservice.messaging.StockClient;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CheckoutService {

    private final StockClient stockClient;

    public StockCheckResponse checkStock(StockCheckRequest request) {
        return stockClient.checkStock(request);
    }
}
```

```java
package br.edu.upf.checkoutservice.controller;

import br.edu.upf.checkoutservice.dto.StockCheckRequest;
import br.edu.upf.checkoutservice.dto.StockCheckResponse;
import br.edu.upf.checkoutservice.service.CheckoutService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/checkout")
@RequiredArgsConstructor
public class CheckoutController {

    private final CheckoutService checkoutService;

    @PostMapping("/stock-check")
    public StockCheckResponse checkStock(@Valid @RequestBody StockCheckRequest request) {
        return checkoutService.checkStock(request);
    }
}
```

Nos dois serviços de RPC, use esta configuração:

```properties
spring.rabbitmq.host=${RABBITMQ_HOST:localhost}
spring.rabbitmq.port=${RABBITMQ_PORT:5672}
spring.rabbitmq.username=${RABBITMQ_USERNAME:guest}
spring.rabbitmq.password=${RABBITMQ_PASSWORD:guest}

app.rabbitmq.stock-request-queue=stock.request.queue
```

No `checkout-service`, configure também:

```properties
spring.application.name=checkout-service
server.port=${PORT:8083}
```

No `stock-service`, configure:

```properties
spring.application.name=stock-service
server.port=${PORT:8084}
```

Teste:

```bash
curl -X POST http://localhost:8083/api/v1/checkout/stock-check \
  -H "Content-Type: application/json" \
  -d '{"productId":"PROD-100","quantity":2}'
```

Resposta esperada:

```json
{
  "productId": "PROD-100",
  "requestedQuantity": 2,
  "availableQuantity": 10,
  "available": true
}
```

## Código completo

Os arquivos completos foram criados no passo a passo, cada um no momento em que o conceito apareceu.

Para conferir se a estrutura ficou correta, o `orders-service` deve ter:

```text
src/main/java/br/edu/upf/ordersservice/
├── config/RabbitConfig.java
├── controller/OrderController.java
├── dto/CreateOrderRequest.java
├── dto/OrderCreatedEvent.java
├── dto/OrderResponse.java
├── mapper/OrderMapper.java
├── messaging/OrderEventPublisher.java
├── model/Order.java
├── repository/OrderRepository.java
└── service/OrderService.java
```

O `notifications-service` deve ter:

```text
src/main/java/br/edu/upf/notificationsservice/
├── config/RabbitConfig.java
├── dto/OrderCreatedEvent.java
└── messaging/OrderCreatedConsumer.java
```

Na comparação com RPC, o `stock-service` deve ter:

```text
src/main/java/br/edu/upf/stockservice/
├── config/RabbitConfig.java
├── dto/StockCheckRequest.java
├── dto/StockCheckResponse.java
├── messaging/StockRequestConsumer.java
└── service/StockService.java
```

E o `checkout-service` deve ter:

```text
src/main/java/br/edu/upf/checkoutservice/
├── config/RabbitConfig.java
├── controller/CheckoutController.java
├── dto/StockCheckRequest.java
├── dto/StockCheckResponse.java
├── messaging/StockClient.java
└── service/CheckoutService.java
```

## Erros comuns

* Esquecer de subir o RabbitMQ antes da aplicação. Se isso acontecer, a conexão AMQP falha.
* Usar porta errada. A aplicação conversa com RabbitMQ pela `5672`; a interface web usa `15672`.
* Criar exchange com nome diferente no producer e no consumer. Os dois serviços precisam falar da mesma exchange.
* Esquecer a binding. Sem binding, a mensagem chega na exchange, mas não vai para a fila.
* Colocar `@RabbitListener` no controller. Consumer de mensagem não é controller HTTP.
* Publicar evento antes de salvar o pedido. Primeiro garanta que o pedido existe, depois avise o restante do sistema.
* Usar RPC para tudo. Se o serviço precisa apenas avisar que algo aconteceu, prefira evento.
* Esquecer que RPC espera resposta. Se o consumer não estiver rodando, a chamada pode demorar ou falhar.
* Usar DTOs incompatíveis nos dois serviços. O JSON enviado precisa bater com o record recebido.
* Colocar regra de estoque dentro do controller do checkout. O checkout pode pedir a informação, mas a regra de estoque pertence ao `stock-service`.

## Resumo

Mensageria permite que microsserviços conversem sem depender sempre de chamadas HTTP diretas.

Com Publish/Subscribe, o producer publica um evento e não espera resposta.

No nosso exemplo, o `orders-service` salva o pedido no MongoDB e publica `pedido.criado`.

O `notifications-service` escuta a fila e reage depois.

RPC com RabbitMQ também existe, mas volta a fazer uma parte do sistema esperar resposta.

Use eventos quando o sistema precisa avisar que algo aconteceu.

Use RPC somente quando a resposta imediata for realmente necessária.

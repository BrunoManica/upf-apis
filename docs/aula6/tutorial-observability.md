# Tutorial: Observabilidade com Spring Boot, Actuator, Micrometer e Prometheus

## Objetivo da aula

Agora que a parte teórica já explicou logs, métricas, traces, health checks e monitoramento, vamos colocar a primeira parte disso em prática usando Java 17 e Spring Boot.

Vamos construir uma API simples de pedidos e adicionar observabilidade básica nela:

* logs com SLF4J;
* health check com Spring Boot Actuator;
* métricas automáticas com Micrometer;
* endpoint Prometheus para coleta de métricas;
* Prometheus rodando em Docker para buscar essas métricas.

O objetivo não é criar uma plataforma completa de observabilidade. Vamos começar pelo que faz sentido para uma primeira prática: a aplicação precisa dizer se está saudável, registrar eventos importantes e expor métricas que possam ser coletadas.

## Resultado final

Ao terminar, teremos:

```text
observability-api -> http://localhost:8080
MongoDB           -> localhost:27017
Prometheus        -> http://localhost:9090
Swagger UI        -> http://localhost:8080/swagger-ui.html
Actuator health   -> http://localhost:8080/actuator/health
Actuator metrics  -> http://localhost:8080/actuator/metrics
Prometheus scrape -> http://localhost:8080/actuator/prometheus
```

A API terá estes endpoints:

```text
POST /api/v1/orders
GET  /api/v1/orders
GET  /api/v1/orders/{id}
```

A estrutura principal ficará assim:

```text
observability-api/
├── src/
│   └── main/
│       ├── java/
│       │   └── br/edu/upf/observabilityapi/
│       │       ├── ObservabilityApiApplication.java
│       │       ├── controller/
│       │       │   └── OrderController.java
│       │       ├── dto/
│       │       │   ├── CreateOrderRequest.java
│       │       │   └── OrderResponse.java
│       │       ├── mapper/
│       │       │   └── OrderMapper.java
│       │       ├── model/
│       │       │   └── Order.java
│       │       ├── repository/
│       │       │   └── OrderRepository.java
│       │       └── service/
│       │           └── OrderService.java
│       └── resources/
│           └── application.properties
├── build.gradle
├── docker-compose.yml
└── prometheus.yml
```

## Contexto

Uma API pode estar funcionando no seu computador e mesmo assim ser difícil de acompanhar quando vai para um ambiente real.

Quando a aplicação roda em produção, não basta saber que ela subiu. Precisamos responder perguntas como:

* a aplicação está saudável?
* o MongoDB está acessível?
* quantas requisições estão chegando?
* quais endpoints estão sendo mais chamados?
* quanto tempo as requisições estão levando?
* os erros aumentaram depois de uma alteração?

O Spring Boot ajuda bastante nesse início porque o Actuator expõe informações operacionais da aplicação. O Micrometer organiza métricas em um formato que outras ferramentas conseguem entender. O Prometheus coleta essas métricas periodicamente.

Por baixo dos panos, a ideia é esta:

```text
Cliente chama a API
        |
        v
Spring Boot processa a requisição
        |
        v
Actuator e Micrometer registram saúde e métricas
        |
        v
Prometheus coleta /actuator/prometheus
```

## Explicação conceitual

Antes do código, vale separar as peças.

`Spring Boot Actuator` adiciona endpoints operacionais à aplicação. Esses endpoints não fazem parte da regra de negócio de pedidos. Eles existem para mostrar informações sobre a aplicação em execução.

`Health check` responde se a aplicação está saudável. Quando usamos MongoDB, o Actuator consegue indicar se a conexão com o banco está funcionando.

`Micrometer` é a camada de métricas usada pelo Spring Boot. Ele mede informações como quantidade de requisições, tempo de resposta, uso de memória e comportamento da JVM.

`Prometheus` é uma ferramenta externa. Ele entra na aplicação periodicamente, lê o endpoint `/actuator/prometheus` e guarda essas métricas ao longo do tempo.

`Logs` registram eventos importantes. Nesta aula, vamos usar SLF4J, que já é o padrão no ecossistema Spring Boot. Em vez de espalhar `System.out.println`, usamos um logger com contexto e nível de severidade.

## Setup inicial

Você precisa ter instalado:

* Java JDK 17;
* Docker e Docker Compose;
* um editor de código;
* um terminal;
* Postman, Insomnia ou curl para testar a API.

No Windows, use `gradlew.bat`.

No Linux, use `./gradlew`.

Use caminhos genéricos nos comandos:

```bash
cd /caminho/para/a/pasta-do-projeto
```

## Passo a passo

### 1. Criar o projeto no Spring Initializr

Abra o Spring Initializr no navegador:

```text
https://start.spring.io/
```

Configure o projeto assim:

```text
Project: Gradle - Groovy
Language: Java
Spring Boot: versão estável 3.x
Group: br.edu.upf
Artifact: observability-api
Name: observability-api
Package name: br.edu.upf.observabilityapi
Packaging: Jar
Java: 17
```

Adicione estas dependências:

* Spring Web;
* Spring Data MongoDB;
* Validation;
* Spring Boot Actuator;
* Lombok.

Depois clique em `Generate`, baixe o arquivo `.zip`, descompacte e abra a pasta no editor.

Entre na pasta do projeto:

```bash
cd /caminho/para/observability-api
```

O Gradle Wrapper já vem no projeto. Isso evita depender de uma instalação global do Gradle.

### 2. Ajustar o build.gradle

Abra o arquivo `build.gradle` e confira se ele está parecido com este:

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
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.14'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

O Spring Initializr já adiciona boa parte disso. O que estamos garantindo aqui é a presença de três dependências importantes para esta aula:

* `spring-boot-starter-actuator`, para health checks e endpoints operacionais;
* `micrometer-registry-prometheus`, para expor métricas no formato Prometheus;
* `springdoc-openapi`, para testar a API pelo Swagger.

### 3. Configurar a aplicação

Edite `src/main/resources/application.properties`:

```properties
spring.application.name=observability-api
server.port=${PORT:8080}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/observability_api}

springdoc.swagger-ui.path=/swagger-ui.html

management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.metrics.tags.application=${spring.application.name}
```

`spring.application.name` dá um nome para a aplicação. Esse nome também aparece como tag nas métricas.

`spring.data.mongodb.uri` usa uma variável de ambiente quando ela existir. Se não existir, a API conecta no MongoDB local.

`management.endpoints.web.exposure.include` define quais endpoints do Actuator ficam disponíveis pela web. Por padrão, o Spring Boot não expõe tudo, e isso é bom. Em uma aplicação real, expor endpoint operacional exige cuidado.

`management.endpoint.health.show-details=always` mostra detalhes do health check. Para aula isso ajuda, porque conseguimos ver se o MongoDB está saudável. Em produção, essa configuração precisa ser avaliada com cuidado para não expor informação demais.

### 4. Criar o Docker Compose

Na raiz do projeto, crie o arquivo `docker-compose.yml`:

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: observability-mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - observability_mongodb_data:/data/db

  prometheus:
    image: prom/prometheus:v2.55.1
    container_name: observability-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  observability_mongodb_data:
```

O MongoDB será usado pela API de pedidos.

O Prometheus vai coletar as métricas expostas pela aplicação.

O endereço `host.docker.internal` permite que o container do Prometheus acesse a aplicação rodando na máquina local. No Docker Desktop isso costuma funcionar automaticamente. No Linux, a linha `extra_hosts` ajuda o Docker a resolver esse nome.

### 5. Criar a configuração do Prometheus

Na raiz do projeto, crie o arquivo `prometheus.yml`:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: observability-api
    metrics_path: /actuator/prometheus
    static_configs:
      - targets:
          - host.docker.internal:8080
```

O Prometheus trabalha com coleta periódica.

A cada cinco segundos, ele chama:

```text
http://host.docker.internal:8080/actuator/prometheus
```

Esse endpoint é gerado pelo Spring Boot Actuator junto com o Micrometer.

### 6. Subir MongoDB e Prometheus

Execute:

```bash
docker compose up -d
```

Verifique se os containers subiram:

```bash
docker compose ps
```

Se precisar ver logs:

```bash
docker compose logs mongodb
docker compose logs prometheus
```

### 7. Criar o model Order

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/model/Order.java`:

```java
package br.edu.upf.observabilityapi.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

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
    private LocalDateTime createdAt;
}
```

`@Document` indica que essa classe representa um documento no MongoDB.

`@Id` marca o campo usado como identificador.

O Lombok reduz código repetitivo. Sem ele, precisaríamos escrever getters, setters, construtores e builder manualmente.

### 8. Criar os DTOs

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/dto/CreateOrderRequest.java`:

```java
package br.edu.upf.observabilityapi.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import lombok.Builder;

@Builder
public record CreateOrderRequest(
        @NotBlank(message = "O nome do cliente é obrigatório")
        String customerName,

        @NotBlank(message = "O email do cliente é obrigatório")
        @Email(message = "O email do cliente deve ser válido")
        String customerEmail,

        @NotNull(message = "O total do pedido é obrigatório")
        @DecimalMin(value = "0.01", message = "O total do pedido deve ser maior que zero")
        BigDecimal total
) {
}
```

Esse DTO representa os dados que entram na API.

As anotações de validação protegem o controller contra dados inválidos. O controller recebe o corpo da requisição e o Spring valida antes de chamar o service.

Agora crie `src/main/java/br/edu/upf/observabilityapi/dto/OrderResponse.java`:

```java
package br.edu.upf.observabilityapi.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import lombok.Builder;

@Builder
public record OrderResponse(
        String id,
        String customerName,
        String customerEmail,
        BigDecimal total,
        LocalDateTime createdAt
) {
}
```

Esse DTO representa os dados que saem da API.

Mesmo que o model seja parecido, não é uma boa prática expor automaticamente tudo que está no documento do MongoDB. O DTO de resposta deixa claro o contrato da API.

### 9. Criar o mapper

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/mapper/OrderMapper.java`:

```java
package br.edu.upf.observabilityapi.mapper;

import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.model.Order;

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
}
```

O mapper separa a conversão entre model e DTO.

Essa conversão não deve ficar no controller. O controller precisa cuidar da entrada HTTP e da resposta HTTP. Ele não deve carregar detalhes de transformação de objetos.

### 10. Criar o repository

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/repository/OrderRepository.java`:

```java
package br.edu.upf.observabilityapi.repository;

import br.edu.upf.observabilityapi.model.Order;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface OrderRepository extends MongoRepository<Order, String> {
}
```

O `MongoRepository` já entrega operações como salvar, listar e buscar por id.

Por baixo dos panos, o Spring Data cria a implementação em tempo de execução. Por isso, aqui declaramos uma interface e não uma classe concreta.

### 11. Criar o service com logs

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/service/OrderService.java`:

```java
package br.edu.upf.observabilityapi.service;

import br.edu.upf.observabilityapi.dto.CreateOrderRequest;
import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.mapper.OrderMapper;
import br.edu.upf.observabilityapi.model.Order;
import br.edu.upf.observabilityapi.repository.OrderRepository;
import java.time.LocalDateTime;
import java.util.List;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
@RequiredArgsConstructor
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;

    public OrderResponse create(CreateOrderRequest request) {
        log.info("Criando pedido para o cliente {}", request.customerEmail());

        Order order = Order.builder()
                .customerName(request.customerName())
                .customerEmail(request.customerEmail())
                .total(request.total())
                .createdAt(LocalDateTime.now())
                .build();

        Order savedOrder = orderRepository.save(order);

        log.info("Pedido criado com sucesso. orderId={}", savedOrder.getId());

        return OrderMapper.mapFrom(savedOrder);
    }

    public List<OrderResponse> findAll() {
        log.info("Listando pedidos");

        return orderRepository.findAll()
                .stream()
                .map(OrderMapper::mapFrom)
                .toList();
    }

    public OrderResponse findById(String id) {
        log.info("Buscando pedido por id. orderId={}", id);

        return orderRepository.findById(id)
                .map(OrderMapper::mapFrom)
                .orElseThrow(() -> {
                    log.warn("Pedido não encontrado. orderId={}", id);
                    return new ResponseStatusException(HttpStatus.NOT_FOUND, "Pedido não encontrado");
                });
    }
}
```

Aqui aparece o primeiro ponto prático de observabilidade: logs úteis.

O service registra quando uma operação importante começa, quando termina e quando algo esperado não acontece, como buscar um pedido inexistente.

Observe que não registramos tudo. Não faz sentido criar log para cada linha. O log precisa ajudar a investigar.

Também evitamos registrar dados sensíveis. O email aparece aqui por ser didático e simples, mas em sistemas reais é preciso avaliar privacidade e regras da empresa.

### 12. Criar o controller

Crie o arquivo `src/main/java/br/edu/upf/observabilityapi/controller/OrderController.java`:

```java
package br.edu.upf.observabilityapi.controller;

import br.edu.upf.observabilityapi.dto.CreateOrderRequest;
import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.service.OrderService;
import jakarta.validation.Valid;
import java.util.List;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

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

    @GetMapping("/{id}")
    public OrderResponse findById(@PathVariable String id) {
        return orderService.findById(id);
    }
}
```

O controller está fino.

Ele recebe HTTP, valida o corpo da requisição com `@Valid`, delega para o service e devolve a resposta.

Ele não acessa repository, não monta objeto manualmente e não coloca regra de negócio no endpoint.

### 13. Conferir a classe principal

O Spring Initializr já cria a classe principal. Confira se ela está assim em `src/main/java/br/edu/upf/observabilityapi/ObservabilityApiApplication.java`:

```java
package br.edu.upf.observabilityapi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ObservabilityApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ObservabilityApiApplication.class, args);
    }
}
```

`@SpringBootApplication` habilita a configuração automática do Spring Boot e a varredura dos componentes do projeto.

Por isso o Spring encontra o controller, o service e o repository sem precisarmos registrar cada classe manualmente.

### 14. Rodar a aplicação

No Linux:

```bash
./gradlew bootRun
```

No Windows:

```bash
gradlew.bat bootRun
```

Quando a aplicação subir, acesse:

```text
http://localhost:8080/swagger-ui.html
```

Agora a API já está pronta para receber pedidos.

### 15. Criar pedidos para gerar comportamento

Crie um pedido:

```bash
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Ana Souza","customerEmail":"ana@email.com","total":149.90}'
```

Liste os pedidos:

```bash
curl http://localhost:8080/api/v1/orders
```

Busque um pedido por id:

```bash
curl http://localhost:8080/api/v1/orders/ID_DO_PEDIDO
```

Também faça uma chamada com erro para gerar métrica de `404`:

```bash
curl -i http://localhost:8080/api/v1/orders/id-inexistente
```

Essas chamadas são importantes porque métricas não aparecem do nada. A aplicação precisa receber tráfego para gerar dados.

### 16. Verificar o health check

Acesse:

```bash
curl http://localhost:8080/actuator/health
```

A resposta deve indicar que a aplicação está `UP`.

Como usamos Spring Data MongoDB, o Actuator também consegue verificar a conexão com o MongoDB.

Uma resposta saudável fica parecida com isto:

```json
{
  "status": "UP",
  "components": {
    "mongo": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

Se o MongoDB estiver parado, esse endpoint ajuda a enxergar o problema rapidamente.

Esse é o papel do health check: mostrar se a aplicação está em condição de funcionar.

### 17. Verificar métricas pelo Actuator

Veja a lista de métricas disponíveis:

```bash
curl http://localhost:8080/actuator/metrics
```

Você verá nomes como:

```text
http.server.requests
jvm.memory.used
process.cpu.usage
application.started.time
```

Agora consulte a métrica de requisições HTTP:

```bash
curl http://localhost:8080/actuator/metrics/http.server.requests
```

Essa métrica mostra informações sobre chamadas HTTP recebidas pela aplicação.

O Spring Boot coleta isso automaticamente porque estamos usando Spring Web, Actuator e Micrometer.

### 18. Verificar o endpoint Prometheus

Acesse:

```bash
curl http://localhost:8080/actuator/prometheus
```

A resposta vem em texto puro, no formato esperado pelo Prometheus.

Procure por métricas HTTP:

```bash
curl http://localhost:8080/actuator/prometheus | grep http_server_requests
```

No Windows PowerShell, use:

```bash
curl http://localhost:8080/actuator/prometheus
```

Depois procure pelo texto `http_server_requests` na saída.

O ponto importante é entender que `/actuator/metrics` é bom para inspecionar métricas pelo próprio Actuator, enquanto `/actuator/prometheus` existe para ferramentas externas coletarem os dados.

### 19. Abrir o Prometheus

Acesse no navegador:

```text
http://localhost:9090
```

Na tela do Prometheus, pesquise por:

```text
http_server_requests_seconds_count
```

Essa métrica indica a quantidade de requisições HTTP registradas.

Pesquise também:

```text
jvm_memory_used_bytes
```

Essa métrica mostra uso de memória da JVM.

Se a aplicação estiver rodando e o Prometheus estiver conseguindo coletar, essas métricas aparecem na interface.

### 20. Conferir se o Prometheus está coletando

No Prometheus, acesse:

```text
Status -> Targets
```

O target `observability-api` deve aparecer como `UP`.

Se aparecer como `DOWN`, verifique:

* se a API está rodando na porta `8080`;
* se o endpoint `/actuator/prometheus` responde no navegador;
* se o arquivo `prometheus.yml` está na raiz do projeto;
* se o Docker Compose foi iniciado depois da criação do arquivo;
* se o Docker consegue resolver `host.docker.internal`.

### 21. Observar os logs

Com a aplicação rodando, faça algumas chamadas para a API.

No terminal onde o `bootRun` está executando, você verá mensagens geradas pelo service.

Elas ajudam a entender o que aconteceu:

```text
Criando pedido para o cliente ana@email.com
Pedido criado com sucesso. orderId=...
Listando pedidos
Buscando pedido por id. orderId=...
Pedido não encontrado. orderId=id-inexistente
```

Esses logs são simples, mas já mostram uma boa prática: registrar eventos relevantes com contexto.

O log não substitui métrica. A métrica mostra o padrão geral. O log ajuda a investigar um caso específico.

## Código completo

### build.gradle

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
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.14'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### application.properties

```properties
spring.application.name=observability-api
server.port=${PORT:8080}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/observability_api}

springdoc.swagger-ui.path=/swagger-ui.html

management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.metrics.tags.application=${spring.application.name}
```

### docker-compose.yml

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: observability-mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - observability_mongodb_data:/data/db

  prometheus:
    image: prom/prometheus:v2.55.1
    container_name: observability-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  observability_mongodb_data:
```

### prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: observability-api
    metrics_path: /actuator/prometheus
    static_configs:
      - targets:
          - host.docker.internal:8080
```

### ObservabilityApiApplication.java

```java
package br.edu.upf.observabilityapi;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ObservabilityApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ObservabilityApiApplication.class, args);
    }
}
```

### Order.java

```java
package br.edu.upf.observabilityapi.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

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
    private LocalDateTime createdAt;
}
```

### CreateOrderRequest.java

```java
package br.edu.upf.observabilityapi.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;
import lombok.Builder;

@Builder
public record CreateOrderRequest(
        @NotBlank(message = "O nome do cliente é obrigatório")
        String customerName,

        @NotBlank(message = "O email do cliente é obrigatório")
        @Email(message = "O email do cliente deve ser válido")
        String customerEmail,

        @NotNull(message = "O total do pedido é obrigatório")
        @DecimalMin(value = "0.01", message = "O total do pedido deve ser maior que zero")
        BigDecimal total
) {
}
```

### OrderResponse.java

```java
package br.edu.upf.observabilityapi.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import lombok.Builder;

@Builder
public record OrderResponse(
        String id,
        String customerName,
        String customerEmail,
        BigDecimal total,
        LocalDateTime createdAt
) {
}
```

### OrderMapper.java

```java
package br.edu.upf.observabilityapi.mapper;

import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.model.Order;

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
}
```

### OrderRepository.java

```java
package br.edu.upf.observabilityapi.repository;

import br.edu.upf.observabilityapi.model.Order;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface OrderRepository extends MongoRepository<Order, String> {
}
```

### OrderService.java

```java
package br.edu.upf.observabilityapi.service;

import br.edu.upf.observabilityapi.dto.CreateOrderRequest;
import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.mapper.OrderMapper;
import br.edu.upf.observabilityapi.model.Order;
import br.edu.upf.observabilityapi.repository.OrderRepository;
import java.time.LocalDateTime;
import java.util.List;
import lombok.RequiredArgsConstructor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
@RequiredArgsConstructor
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;

    public OrderResponse create(CreateOrderRequest request) {
        log.info("Criando pedido para o cliente {}", request.customerEmail());

        Order order = Order.builder()
                .customerName(request.customerName())
                .customerEmail(request.customerEmail())
                .total(request.total())
                .createdAt(LocalDateTime.now())
                .build();

        Order savedOrder = orderRepository.save(order);

        log.info("Pedido criado com sucesso. orderId={}", savedOrder.getId());

        return OrderMapper.mapFrom(savedOrder);
    }

    public List<OrderResponse> findAll() {
        log.info("Listando pedidos");

        return orderRepository.findAll()
                .stream()
                .map(OrderMapper::mapFrom)
                .toList();
    }

    public OrderResponse findById(String id) {
        log.info("Buscando pedido por id. orderId={}", id);

        return orderRepository.findById(id)
                .map(OrderMapper::mapFrom)
                .orElseThrow(() -> {
                    log.warn("Pedido não encontrado. orderId={}", id);
                    return new ResponseStatusException(HttpStatus.NOT_FOUND, "Pedido não encontrado");
                });
    }
}
```

### OrderController.java

```java
package br.edu.upf.observabilityapi.controller;

import br.edu.upf.observabilityapi.dto.CreateOrderRequest;
import br.edu.upf.observabilityapi.dto.OrderResponse;
import br.edu.upf.observabilityapi.service.OrderService;
import jakarta.validation.Valid;
import java.util.List;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

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

    @GetMapping("/{id}")
    public OrderResponse findById(@PathVariable String id) {
        return orderService.findById(id);
    }
}
```

## Erros comuns

### Esquecer a dependência do Prometheus

Sem `micrometer-registry-prometheus`, o endpoint `/actuator/prometheus` não aparece.

O Actuator cria endpoints operacionais, mas o formato Prometheus depende do registry específico.

### A aplicação sobe, mas o health check do MongoDB fica DOWN

Isso normalmente significa que o MongoDB não está rodando ou que a URI está errada.

Confira se o container está ativo:

```bash
docker compose ps
```

Depois confira a configuração:

```properties
spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/observability_api}
```

### Prometheus mostra o target como DOWN

Primeiro teste o endpoint diretamente:

```bash
curl http://localhost:8080/actuator/prometheus
```

Se esse endpoint não responder, o problema está na aplicação ou na configuração do Actuator.

Se ele responder no navegador, mas o Prometheus continuar `DOWN`, revise o `prometheus.yml` e reinicie o Docker Compose.

### Esperar métricas sem gerar requisições

Algumas métricas só aparecem depois que a aplicação executa uma operação.

Faça chamadas para criar, listar e buscar pedidos antes de procurar métricas HTTP.

### Expor endpoints demais do Actuator

Nesta aula expomos apenas:

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Evite expor todos os endpoints com `*` sem entender o impacto. Alguns endpoints podem revelar detalhes internos da aplicação.

### Colocar regra de negócio no controller

O controller não deve salvar diretamente no MongoDB nem montar manualmente a resposta.

Neste tutorial, o controller delega para o service. O service coordena a regra. O repository conversa com o banco. O mapper converte model para DTO.

## Resumo

Neste tutorial, criamos uma API Spring Boot simples e adicionamos observabilidade básica.

O Actuator expôs endpoints operacionais, como health check e métricas.

O Micrometer organizou métricas da aplicação.

O Prometheus coletou essas métricas pelo endpoint `/actuator/prometheus`.

Os logs com SLF4J deram contexto para operações importantes da API.

Esse é um começo sólido. Antes de avançar para tracing distribuído e dashboards mais completos, é importante dominar essa base: health, logs e métricas.

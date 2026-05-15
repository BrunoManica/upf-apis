# Tutorial: Construindo uma Payment API com Java 17

## Objetivo da aula

Nesta aula vamos construir uma API de pagamentos com Java 17, Spring Boot e MongoDB.

Vamos fazer isso em dois momentos:

- primeiro, uma versão `legacy-payment`, funcionando, mas ignorando SOLID de propósito
- depois, uma versão `payments`, refatorada com responsabilidades mais claras

Essa comparação é importante. Fica muito mais fácil entender SOLID quando primeiro sentimos o problema de um código que cresceu sem organização.

## Resultado final

Ao final, teremos uma API com:

- criação de pagamentos
- listagem de pagamentos
- busca por id
- estatísticas simples
- MongoDB rodando via Docker
- Swagger UI para testar os endpoints
- exemplo legado sem SOLID
- exemplo melhorado com Strategy e injeção de dependência

Endpoints principais:

```text
POST /api/v1/legacy-payments
GET  /api/v1/legacy-payments
GET  /api/v1/legacy-payments/{id}
GET  /api/v1/legacy-payments/stats

POST /api/v1/payments
GET  /api/v1/payments
GET  /api/v1/payments/{id}
GET  /api/v1/payments/stats
```

## Contexto

Pagamentos são um ótimo exemplo para estudar SOLID porque cada forma de pagamento tem detalhes próprios.

PIX precisa de chave e QR Code. Cartão precisa de número, CVV, validade e uma taxa percentual. Boleto precisa de vencimento e número do boleto.

Se colocarmos tudo isso dentro de um único service, o código até funciona, mas começa a ficar difícil de alterar. Quando entrar um novo tipo de pagamento, a tendência será abrir a mesma classe e adicionar mais um `if`.

Nesta aula vamos fazer isso primeiro, justamente para enxergar o problema. Depois vamos melhorar.

## Explicação conceitual

A aplicação vai seguir o padrão mais comum para APIs Spring Boot:

```text
controller -> service -> repository -> MongoDB
```

Cada parte tem uma função:

- controller recebe a requisição HTTP
- DTO representa os dados de entrada e saída
- service concentra regra de negócio
- repository acessa o MongoDB
- model representa o documento salvo no banco

Na versão legacy, o service vai misturar responsabilidades. Na versão com SOLID, vamos separar o processamento de cada tipo de pagamento em classes próprias.

## Setup inicial

### Pré-requisitos

- Java 17 instalado
- Docker e Docker Compose
- acesso a um terminal no Windows, Linux ou macOS
- editor de código, como IntelliJ IDEA ou VS Code

### Criar o projeto Spring Boot

Acesse o Spring Initializr:

```text
https://start.spring.io
```

Use estas opções:

- Project: Gradle - Groovy
- Language: Java
- Spring Boot: versão estável sugerida pelo site
- Group: `br.edu.upf`
- Artifact: `payment-api`
- Name: `payment-api`
- Description: API de pagamentos com Spring Boot e MongoDB
- Package name: `br.edu.upf.paymentapi`
- Packaging: Jar
- Java: 17

Para evitar diferença entre o que o Spring Initializr gera e o que vamos usar no código, adicione somente:

- Spring Web.

Depois clique em `Generate` para baixar o `.zip`.

O ponto importante é este: vamos usar o Spring Initializr apenas para criar a estrutura inicial do projeto e o Gradle Wrapper. As outras bibliotecas serão colocadas manualmente no `build.gradle`, do mesmo jeito que fizemos na aula 2.

### Descompactar e abrir o projeto

No terminal, vá para a pasta onde o arquivo foi baixado e execute:

```bash
unzip payment-api.zip -d payment-api
cd payment-api
```

No Windows, você também pode descompactar pelo explorador de arquivos e depois abrir a pasta `payment-api` no editor.

### Configurar a aplicação

Abra `src/main/resources/application.properties`:

```properties
spring.application.name=payment-api
server.port=${PORT:8080}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://payment_user:payment_pass@localhost:27017/payment_db?authSource=admin}

springdoc.swagger-ui.path=/swagger-ui.html
```

Essa configuração deixa a API preparada para rodar localmente e também em container.

`server.port=${PORT:8080}` usa a variável `PORT` quando ela existir. Se não existir, a API sobe na porta `8080`.

`spring.data.mongodb.uri` aponta para o MongoDB local por padrão. A variável `MONGODB_URI` permite trocar a conexão sem alterar código.

`springdoc.swagger-ui.path` define o endereço da tela do Swagger.

### Substituir o build.gradle

Abra o arquivo `build.gradle`, apague o conteúdo atual e substitua por este:

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '4.0.6'
	id 'io.spring.dependency-management' version '1.1.7'
}

group = 'br.edu.upf'
version = '0.0.1-SNAPSHOT'

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
	implementation 'org.springframework.boot:spring-boot-starter-validation'
	implementation 'org.springframework.boot:spring-boot-starter-webmvc'
	implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.9'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-data-mongodb-test'
	testImplementation 'org.springframework.boot:spring-boot-starter-validation-test'
	testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
	testCompileOnly 'org.projectlombok:lombok'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
	testAnnotationProcessor 'org.projectlombok:lombok'
}

tasks.named('test') {
	useJUnitPlatform()
}
```

Mesmo tendo marcado apenas `Spring Web` no Initializr, agora o projeto passa a ter MongoDB, validação, Swagger e Lombok porque colocamos essas bibliotecas manualmente no Gradle.

A comunicação com o MongoDB continua existindo. Ela não vem da opção marcada no Initializr; ela vem desta dependência:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
```

Esse starter adiciona o Spring Data MongoDB ao projeto. Mais adiante, quando criarmos o `PaymentRepository`, o Spring vai usar essa biblioteca para transformar chamadas Java, como `save`, `findAll` e `findById`, em operações no MongoDB.

Esse arquivo usa o starter `spring-boot-starter-webmvc`, que é o nome gerado nas versões novas do Spring Boot. Se o editor pedir para recarregar o projeto Gradle, aceite.

### Configurar informações do Swagger

Crie a pasta:

```text
src/main/java/br/edu/upf/paymentapi/config
```

Crie o arquivo `src/main/java/br/edu/upf/paymentapi/config/OpenApiConfig.java`:

```java
package br.edu.upf.paymentapi.config;

import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import org.springframework.context.annotation.Configuration;

@Configuration
@OpenAPIDefinition(
        info = @Info(
                title = "Payment API",
                version = "1.0",
                description = "API de pagamentos com Spring Boot, MongoDB e SOLID"
        )
)
public class OpenApiConfig {
}
```

`@Configuration` diz ao Spring que essa classe participa da configuração da aplicação.

`@OpenAPIDefinition` personaliza as informações que aparecem no Swagger.

## Passo a passo

## Passo 1: Configurar o MongoDB

### 1.1 Criar o Docker Compose

Na raiz do projeto, crie o arquivo `docker-compose.yml`:

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: payment-mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: payment_user
      MONGO_INITDB_ROOT_PASSWORD: payment_pass
      MONGO_INITDB_DATABASE: payment_db
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

Esse container sobe um MongoDB local na porta `27017`.

A API vai se conectar nesse banco usando a URI definida em `application.properties`. O container é o banco de dados rodando localmente; o `spring-boot-starter-data-mongodb` é a biblioteca que permite a aplicação Java conversar com ele.

### 1.2 Subir o banco

```bash
docker compose up -d
```

Para verificar:

```bash
docker ps
```

Para parar:

```bash
docker compose down
```

## Passo 2: Criar o model e o repository

### 2.1 Criar o documento Payment

Crie `src/main/java/br/edu/upf/paymentapi/model/Payment.java`:

```java
package br.edu.upf.paymentapi.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import lombok.Builder;
import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@Builder
@Document(collection = "payments")
public class Payment {

    @Id
    private String id;
    private String type;
    private BigDecimal amount;
    private String status;
    private String customerName;
    private String customerEmail;
    private Map<String, Object> metadata = new HashMap<>();
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

`@Document` informa ao Spring Data MongoDB que essa classe representa documentos da coleção `payments`.

`@Data` vem do Lombok e gera getters, setters, `equals`, `hashCode` e `toString` em tempo de compilação. Como nenhum campo é `final` ou `@NonNull`, o `@Data` também gera um construtor sem argumentos, o que o Spring Data MongoDB precisa para reconstruir objetos ao ler o banco.

`@Builder` gera uma API de construção fluente para essa classe. Em vez de criar o objeto e chamar vários setters um a um, o código que monta o `Payment` vai poder usar `Payment.builder().type(...).amount(...).build()`. Isso vai ficar evidente quando criarmos o `PaymentMapper` mais adiante.

Uma coisa importante: evite colocar lógica de negócio dentro do `Payment` ou de qualquer DTO. Esse objeto representa apenas o documento salvo no banco. Cálculos, validações e transformações pertencem às camadas de serviço.

### 2.2 Criar o repository

Crie `src/main/java/br/edu/upf/paymentapi/repository/PaymentRepository.java`:

```java
package br.edu.upf.paymentapi.repository;

import br.edu.upf.paymentapi.model.Payment;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface PaymentRepository extends MongoRepository<Payment, String> {
}
```

Não precisamos implementar essa interface. O Spring Data cria a implementação em tempo de execução.

É aqui que a comunicação com o MongoDB fica visível no código. Quando o service chamar `paymentRepository.save(payment)`, o Spring Data MongoDB vai gravar um documento na coleção `payments`. Quando chamar `findAll()` ou `findById(id)`, ele vai consultar essa mesma coleção.

## Passo 3: Criar DTOs compartilhados

### 3.1 Request de criação

Crie `src/main/java/br/edu/upf/paymentapi/dto/CreatePaymentRequest.java`:

```java
package br.edu.upf.paymentapi.dto;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public record CreatePaymentRequest(
        @NotBlank String type,
        @NotNull @DecimalMin("0.01") BigDecimal amount,
        @NotBlank String customerName,
        @NotBlank @Email String customerEmail,
        String cardNumber,
        String cvv,
        String expiryDate,
        String pixKey,
        String dueDate
) {
}
```

Usamos `record` porque DTO é só transporte de dados. Ele não precisa ter regra de negócio.

### 3.2 Response de pagamento

Crie `src/main/java/br/edu/upf/paymentapi/dto/PaymentResponse.java`:

```java
package br.edu.upf.paymentapi.dto;

import br.edu.upf.paymentapi.model.Payment;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

public record PaymentResponse(
        String id,
        String type,
        BigDecimal amount,
        String status,
        String customerName,
        String customerEmail,
        Map<String, Object> metadata,
        LocalDateTime createdAt,
        LocalDateTime updatedAt
) {

    public static PaymentResponse from(Payment payment) {
        return new PaymentResponse(
                payment.getId(),
                payment.getType(),
                payment.getAmount(),
                payment.getStatus(),
                payment.getCustomerName(),
                payment.getCustomerEmail(),
                payment.getMetadata(),
                payment.getCreatedAt(),
                payment.getUpdatedAt()
        );
    }
}
```

O response evita devolver diretamente o objeto interno da aplicação sem controle.

### 3.3 Response de estatísticas

Crie `src/main/java/br/edu/upf/paymentapi/dto/PaymentStatsResponse.java`:

```java
package br.edu.upf.paymentapi.dto;

import java.math.BigDecimal;
import java.util.Map;

public record PaymentStatsResponse(
        long totalPayments,
        BigDecimal totalAmount,
        Map<String, Long> byType
) {
}
```

## Passo 4: Criar a versão legacy sem SOLID

Agora vamos criar a parte propositalmente ruim.

Ela vai funcionar, mas vai misturar validação, processamento, cálculo de taxa, persistência, log e estatísticas no mesmo service.

### 4.1 Criar o LegacyPaymentService

Crie `src/main/java/br/edu/upf/paymentapi/service/LegacyPaymentService.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.dto.PaymentStatsResponse;
import br.edu.upf.paymentapi.model.Payment;
import br.edu.upf.paymentapi.repository.PaymentRepository;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class LegacyPaymentService {

    private final PaymentRepository paymentRepository;

    public LegacyPaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    public Payment create(CreatePaymentRequest request) {
        validateBasicData(request);

        Map<String, Object> metadata = new HashMap<>();

        if ("PIX".equals(request.type())) {
            if (request.pixKey() == null || request.pixKey().isBlank()) {
                throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Chave PIX e obrigatoria");
            }

            metadata.put("pixKey", request.pixKey());
            metadata.put("qrCode", "pix-qr-code-" + System.currentTimeMillis());
        } else if ("CREDIT_CARD".equals(request.type())) {
            if (request.cardNumber() == null || request.cvv() == null || request.expiryDate() == null) {
                throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Dados do cartao sao obrigatorios");
            }

            metadata.put("cardNumber", maskCard(request.cardNumber()));
            metadata.put("cvv", "***");
            metadata.put("expiryDate", request.expiryDate());
            metadata.put("processor", detectCardProcessor(request.cardNumber()));
        } else if ("BOLETO".equals(request.type())) {
            if (request.dueDate() == null || request.dueDate().isBlank()) {
                throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Data de vencimento e obrigatoria");
            }

            metadata.put("boletoNumber", generateBoletoNumber());
            metadata.put("dueDate", request.dueDate());
        } else {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Tipo de pagamento nao suportado");
        }

        BigDecimal fee = calculateFee(request.amount(), request.type());
        BigDecimal totalAmount = request.amount().add(fee);
        String status = determineStatus(request.amount(), request.type());

        Payment payment = new Payment();
        payment.setType(request.type());
        payment.setAmount(totalAmount);
        payment.setStatus(status);
        payment.setCustomerName(request.customerName().toUpperCase());
        payment.setCustomerEmail(request.customerEmail().toLowerCase());
        payment.setMetadata(metadata);
        payment.setCreatedAt(LocalDateTime.now());
        payment.setUpdatedAt(LocalDateTime.now());

        Payment savedPayment = paymentRepository.save(payment);

        System.out.println("Email enviado para " + savedPayment.getCustomerEmail());
        System.out.println("Pagamento criado com sucesso: " + savedPayment.getId());

        return savedPayment;
    }

    public List<Payment> findAll() {
        return paymentRepository.findAll();
    }

    public Payment findById(String id) {
        return paymentRepository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Pagamento nao encontrado"));
    }

    public PaymentStatsResponse getStats() {
        List<Payment> payments = paymentRepository.findAll();

        BigDecimal totalAmount = payments.stream()
                .map(Payment::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, Long> byType = payments.stream()
                .collect(Collectors.groupingBy(Payment::getType, Collectors.counting()));

        return new PaymentStatsResponse(payments.size(), totalAmount, byType);
    }

    private void validateBasicData(CreatePaymentRequest request) {
        if (request.customerName() == null || request.customerName().isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Nome do cliente e obrigatorio");
        }

        if (request.amount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Valor deve ser maior que zero");
        }
    }

    private BigDecimal calculateFee(BigDecimal amount, String type) {
        if ("CREDIT_CARD".equals(type)) {
            return amount.multiply(new BigDecimal("0.03"));
        }

        if ("BOLETO".equals(type)) {
            return new BigDecimal("2.50");
        }

        if ("CRYPTO".equals(type)) {
            return new BigDecimal("1.50");
        }

        return BigDecimal.ZERO;
    }

    private String determineStatus(BigDecimal amount, String type) {
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return "PENDING";
        }

        if ("CREDIT_CARD".equals(type) && amount.compareTo(new BigDecimal("5000")) > 0) {
            return "PENDING";
        }

        return "APPROVED";
    }

    private String maskCard(String cardNumber) {
        return "************" + cardNumber.substring(cardNumber.length() - 4);
    }

    private String detectCardProcessor(String cardNumber) {
        if (cardNumber.startsWith("4")) {
            return "Visa";
        }

        if (cardNumber.startsWith("5")) {
            return "Mastercard";
        }

        return "Unknown";
    }

    private String generateBoletoNumber() {
        return "34191.79001 01043.510047 91020.150008 1 84460000020000";
    }
}
```

Esse código é útil para aula porque deixa os problemas visíveis:

- muitos motivos para mudar: a mesma classe toca validação, processamento, cálculo de taxa, persistência e notificação
- cada novo tipo de pagamento exige abrir o mesmo service e adicionar mais um `if`, como o `CRYPTO` que foi adicionado depois
- o método `calculateFee` cresce indefinidamente: PIX, BOLETO, CREDIT_CARD, CRYPTO... e por aí vai
- difícil testar um tipo de pagamento isoladamente sem passar por todo o método `create`
- o `System.out.println` para notificação misturado no meio do fluxo principal torna a extração futura ainda mais difícil

### 4.2 Criar o LegacyPaymentController

Crie `src/main/java/br/edu/upf/paymentapi/controller/LegacyPaymentController.java`:

```java
package br.edu.upf.paymentapi.controller;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.dto.PaymentResponse;
import br.edu.upf.paymentapi.dto.PaymentStatsResponse;
import br.edu.upf.paymentapi.service.LegacyPaymentService;
import jakarta.validation.Valid;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/legacy-payments")
public class LegacyPaymentController {

    private final LegacyPaymentService legacyPaymentService;

    public LegacyPaymentController(LegacyPaymentService legacyPaymentService) {
        this.legacyPaymentService = legacyPaymentService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public PaymentResponse create(@RequestBody @Valid CreatePaymentRequest request) {
        return PaymentResponse.from(legacyPaymentService.create(request));
    }

    @GetMapping
    public List<PaymentResponse> findAll() {
        return legacyPaymentService.findAll()
                .stream()
                .map(PaymentResponse::from)
                .toList();
    }

    @GetMapping("/{id}")
    public PaymentResponse findById(@PathVariable String id) {
        return PaymentResponse.from(legacyPaymentService.findById(id));
    }

    @GetMapping("/stats")
    public PaymentStatsResponse getStats() {
        return legacyPaymentService.getStats();
    }
}
```

O controller está aceitável: ele recebe HTTP, valida o body com `@Valid` e delega para o service. O problema principal está no service.

## Passo 5: Criar a versão com SOLID

Agora vamos criar a versão melhorada.

Vamos aplicar três princípios SOLID de forma clara:

- SRP: cada classe com uma única responsabilidade
- OCP: Strategy Pattern para que novos tipos de pagamento entrem sem modificar o service principal
- DIP: o service depende de abstrações, não de implementações

A mudança acontece em três frentes:

- `ValidatePayment` assume a responsabilidade de determinar o status do pagamento
- `PaymentMapper` assume a responsabilidade de construir o objeto `Payment` antes de salvar
- A interface `PaymentProcessor` e suas implementações assumem o processamento específico de cada tipo

### 5.1 Criar ValidatePayment

No `LegacyPaymentService`, a lógica de status estava misturada dentro do mesmo método que também validava dados, calculava taxa e persistia. Agora ela ganha uma classe própria.

Crie `src/main/java/br/edu/upf/paymentapi/service/ValidatePayment.java`:

```java
package br.edu.upf.paymentapi.service;

import java.math.BigDecimal;
import org.springframework.stereotype.Component;

@Component
public class ValidatePayment {

    public String determineStatus(BigDecimal amount, String type) {
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return "PENDING";
        }

        if ("CREDIT_CARD".equals(type) && amount.compareTo(new BigDecimal("5000")) > 0) {
            return "PENDING";
        }

        return "APPROVED";
    }
}
```

Essa classe só faz uma coisa: dado um valor e um tipo, diz qual status o pagamento deve ter.

### 5.2 Criar ProcessedPayment

Esse record representa o resultado do processamento de um pagamento antes de ser salvo.

Crie `src/main/java/br/edu/upf/paymentapi/service/ProcessedPayment.java`:

```java
package br.edu.upf.paymentapi.service;

import java.math.BigDecimal;
import java.util.Map;

public record ProcessedPayment(
        BigDecimal fee,
        Map<String, Object> metadata
) {
}
```

O `fee` sai do record para que o `PaymentMapper` consiga calcular o total sem precisar chamar o processor novamente.

### 5.3 Criar PaymentMapper

No `LegacyPaymentService`, a construção manual do objeto `Payment` estava espalhada no meio do fluxo principal. Agora ela vai para um mapper dedicado.

Crie a pasta:

```text
src/main/java/br/edu/upf/paymentapi/mappers
```

Crie `src/main/java/br/edu/upf/paymentapi/mappers/PaymentMapper.java`:

```java
package br.edu.upf.paymentapi.mappers;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.model.Payment;
import br.edu.upf.paymentapi.service.ProcessedPayment;
import java.time.LocalDateTime;
import org.springframework.stereotype.Component;

@Component
public class PaymentMapper {

    public Payment toPayment(CreatePaymentRequest request, ProcessedPayment processed, String status) {
        return Payment.builder()
                .type(request.type())
                .amount(request.amount().add(processed.fee()))
                .status(status)
                .customerName(request.customerName().toUpperCase())
                .customerEmail(request.customerEmail().toLowerCase())
                .metadata(processed.metadata())
                .createdAt(LocalDateTime.now())
                .updatedAt(LocalDateTime.now())
                .build();
    }
}
```

O mapper recebe três entradas: o request com os dados do cliente, o resultado do processamento com a taxa e os metadados, e o status calculado. Com isso, monta e devolve um `Payment` pronto para persistência.

Usar `Payment.builder()` em vez de criar o objeto e chamar setters um a um tem uma vantagem prática: torna imediatamente visível que o `Payment` está sendo construído com todos os campos necessários num único ponto, sem risco de esquecer de atribuir um campo no meio de outros métodos.

O `PaymentMapper` não decide taxa, não valida tipo, não determina status. Só constrói. Isso é SRP funcionando: uma responsabilidade por classe.

### 5.4 Criar a interface PaymentProcessor

Essa interface é o contrato que cada estratégia de pagamento deve cumprir.

Crie `src/main/java/br/edu/upf/paymentapi/service/PaymentProcessor.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;

public interface PaymentProcessor {

    String getType();

    ProcessedPayment process(CreatePaymentRequest request);
}
```

### 5.5 Criar o processador PIX

Crie `src/main/java/br/edu/upf/paymentapi/service/PixPaymentProcessor.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class PixPaymentProcessor implements PaymentProcessor {

    @Override
    public String getType() {
        return "PIX";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        if (request.pixKey() == null || request.pixKey().isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Chave PIX e obrigatoria");
        }

        return new ProcessedPayment(
                BigDecimal.ZERO,
                Map.of(
                        "pixKey", request.pixKey(),
                        "qrCode", "pix-qr-code-" + System.currentTimeMillis()
                )
        );
    }
}
```

### 5.6 Criar o processador de cartão

Crie `src/main/java/br/edu/upf/paymentapi/service/CreditCardPaymentProcessor.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class CreditCardPaymentProcessor implements PaymentProcessor {

    private static final BigDecimal FEE_RATE = new BigDecimal("0.03");

    @Override
    public String getType() {
        return "CREDIT_CARD";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        if (request.cardNumber() == null || request.cvv() == null || request.expiryDate() == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Dados do cartao sao obrigatorios");
        }

        return new ProcessedPayment(
                request.amount().multiply(FEE_RATE),
                Map.of(
                        "cardNumber", maskCard(request.cardNumber()),
                        "cvv", "***",
                        "expiryDate", request.expiryDate(),
                        "processor", detectCardProcessor(request.cardNumber())
                )
        );
    }

    private String maskCard(String cardNumber) {
        return "************" + cardNumber.substring(cardNumber.length() - 4);
    }

    private String detectCardProcessor(String cardNumber) {
        if (cardNumber.startsWith("4")) {
            return "Visa";
        }

        if (cardNumber.startsWith("5")) {
            return "Mastercard";
        }

        return "Unknown";
    }
}
```

### 5.7 Criar o processador de boleto

Crie `src/main/java/br/edu/upf/paymentapi/service/BoletoPaymentProcessor.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import java.math.BigDecimal;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class BoletoPaymentProcessor implements PaymentProcessor {

    private static final BigDecimal BOLETO_FEE = new BigDecimal("2.50");

    @Override
    public String getType() {
        return "BOLETO";
    }

    @Override
    public ProcessedPayment process(CreatePaymentRequest request) {
        if (request.dueDate() == null || request.dueDate().isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Data de vencimento e obrigatoria");
        }

        return new ProcessedPayment(
                BOLETO_FEE,
                Map.of(
                        "boletoNumber", "34191.79001 01043.510047 91020.150008 1 84460000020000",
                        "dueDate", request.dueDate()
                )
        );
    }
}
```

### 5.8 Criar o resolver de estratégias

Crie `src/main/java/br/edu/upf/paymentapi/service/PaymentProcessorResolver.java`:

```java
package br.edu.upf.paymentapi.service;

import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;

@Component
public class PaymentProcessorResolver {

    private final Map<String, PaymentProcessor> processors;

    public PaymentProcessorResolver(List<PaymentProcessor> processors) {
        this.processors = processors.stream()
                .collect(Collectors.toMap(PaymentProcessor::getType, Function.identity()));
    }

    public PaymentProcessor resolve(String type) {
        PaymentProcessor processor = processors.get(type);

        if (processor == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Tipo de pagamento nao suportado");
        }

        return processor;
    }
}
```

O Spring encontra todos os beans que implementam `PaymentProcessor` e entrega a lista no construtor. O resolver monta um mapa por tipo e entrega o processador certo quando o service pedir.

### 5.9 Criar o PaymentService

Crie `src/main/java/br/edu/upf/paymentapi/service/PaymentService.java`:

```java
package br.edu.upf.paymentapi.service;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.dto.PaymentStatsResponse;
import br.edu.upf.paymentapi.mappers.PaymentMapper;
import br.edu.upf.paymentapi.model.Payment;
import br.edu.upf.paymentapi.repository.PaymentRepository;
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class PaymentService {

    private final PaymentProcessorResolver processorResolver;
    private final PaymentRepository paymentRepository;
    private final PaymentMapper paymentMapper;
    private final ValidatePayment validatePayment;

    public PaymentService(
            PaymentProcessorResolver processorResolver,
            PaymentRepository paymentRepository,
            PaymentMapper paymentMapper,
            ValidatePayment validatePayment
    ) {
        this.processorResolver = processorResolver;
        this.paymentRepository = paymentRepository;
        this.paymentMapper = paymentMapper;
        this.validatePayment = validatePayment;
    }

    public Payment create(CreatePaymentRequest request) {
        PaymentProcessor processor = processorResolver.resolve(request.type());
        ProcessedPayment processed = processor.process(request);

        String status = validatePayment.determineStatus(request.amount(), request.type());
        Payment payment = paymentMapper.toPayment(request, processed, status);

        return paymentRepository.save(payment);
    }

    public List<Payment> findAll() {
        return paymentRepository.findAll();
    }

    public Payment findById(String id) {
        return paymentRepository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Pagamento nao encontrado"));
    }

    public PaymentStatsResponse getStats() {
        List<Payment> payments = paymentRepository.findAll();

        BigDecimal totalAmount = payments.stream()
                .map(Payment::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, Long> byType = payments.stream()
                .collect(Collectors.groupingBy(Payment::getType, Collectors.counting()));

        return new PaymentStatsResponse(payments.size(), totalAmount, byType);
    }
}
```

Compare o `create` agora com o do `LegacyPaymentService`:

```text
// Legacy: 30 linhas misturando if, calculo, persistencia, log
// Refatorado: 4 linhas, cada uma delegando para quem entende do assunto
```

O service não sabe mais como PIX funciona. Não sabe mais calcular taxa de cartão. Não sabe mais como construir um objeto `Payment`. Cada responsabilidade foi para a classe certa.

### 5.10 Criar o PaymentController

Crie `src/main/java/br/edu/upf/paymentapi/controller/PaymentController.java`:

```java
package br.edu.upf.paymentapi.controller;

import br.edu.upf.paymentapi.dto.CreatePaymentRequest;
import br.edu.upf.paymentapi.dto.PaymentResponse;
import br.edu.upf.paymentapi.dto.PaymentStatsResponse;
import br.edu.upf.paymentapi.service.PaymentService;
import jakarta.validation.Valid;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    private final PaymentService paymentService;

    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public PaymentResponse create(@RequestBody @Valid CreatePaymentRequest request) {
        return PaymentResponse.from(paymentService.create(request));
    }

    @GetMapping
    public List<PaymentResponse> findAll() {
        return paymentService.findAll()
                .stream()
                .map(PaymentResponse::from)
                .toList();
    }

    @GetMapping("/{id}")
    public PaymentResponse findById(@PathVariable String id) {
        return PaymentResponse.from(paymentService.findById(id));
    }

    @GetMapping("/stats")
    public PaymentStatsResponse getStats() {
        return paymentService.getStats();
    }
}
```

## Passo 6: Executar e testar a API

### 6.1 Subir o MongoDB

```bash
docker compose up -d
```

### 6.2 Rodar a aplicação

No Windows:

```bash
gradlew.bat bootRun
```

No Linux ou macOS:

```bash
./gradlew bootRun
```

### 6.3 Abrir o Swagger

Acesse:

```text
http://localhost:8080/swagger-ui.html
```

### 6.4 Criar pagamento PIX na versão legacy

```bash
curl -X POST http://localhost:8080/api/v1/legacy-payments \
  -H "Content-Type: application/json" \
  -d '{
    "type": "PIX",
    "amount": 100.50,
    "customerName": "Joao Silva",
    "customerEmail": "joao@email.com",
    "pixKey": "joao@email.com"
  }'
```

### 6.5 Criar pagamento PIX na versão com SOLID

```bash
curl -X POST http://localhost:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "type": "PIX",
    "amount": 100.50,
    "customerName": "Joao Silva",
    "customerEmail": "joao@email.com",
    "pixKey": "joao@email.com"
  }'
```

### 6.6 Criar pagamento com cartão

```bash
curl -X POST http://localhost:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "type": "CREDIT_CARD",
    "amount": 250.00,
    "customerName": "Maria Souza",
    "customerEmail": "maria@email.com",
    "cardNumber": "4111111111111111",
    "cvv": "123",
    "expiryDate": "12/2030"
  }'
```

### 6.7 Criar pagamento com boleto

```bash
curl -X POST http://localhost:8080/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "type": "BOLETO",
    "amount": 80.00,
    "customerName": "Carlos Lima",
    "customerEmail": "carlos@email.com",
    "dueDate": "2026-06-10"
  }'
```

## Código completo

Ao final, a estrutura principal fica assim:

```text
payment-api/
├── build.gradle
├── docker-compose.yml
└── src/main/
    ├── java/br/edu/upf/paymentapi/
    │   ├── PaymentApiApplication.java
    │   ├── config/
    │   │   └── OpenApiConfig.java
    │   ├── controller/
    │   │   ├── LegacyPaymentController.java
    │   │   └── PaymentController.java
    │   ├── dto/
    │   │   ├── CreatePaymentRequest.java
    │   │   ├── PaymentResponse.java
    │   │   └── PaymentStatsResponse.java
    │   ├── mappers/
    │   │   └── PaymentMapper.java
    │   ├── model/
    │   │   └── Payment.java
    │   ├── repository/
    │   │   └── PaymentRepository.java
    │   └── service/
    │       ├── LegacyPaymentService.java
    │       ├── PaymentService.java
    │       ├── PaymentProcessor.java
    │       ├── PaymentProcessorResolver.java
    │       ├── ProcessedPayment.java
    │       ├── ValidatePayment.java
    │       ├── PixPaymentProcessor.java
    │       ├── CreditCardPaymentProcessor.java
    │       └── BoletoPaymentProcessor.java
    └── resources/
        └── application.properties
```

## Erros comuns

- Esquecer de subir o MongoDB antes de rodar a aplicação.
- Usar `localhost` dentro de outro container. Dentro de containers, o host precisa ser o nome do serviço no Docker Compose.
- Esquecer `authSource=admin` na URI do MongoDB quando usar usuário root.
- Colocar regra de negócio no controller.
- Criar strategy para tudo antes de existir variação real.
- Usar `@Autowired` em atributo em vez de construtor.
- Alterar o service legacy para ficar bom antes da comparação. Ele deve ficar ruim de propósito para a aula fazer sentido.
- Colocar lógica de negócio dentro de um DTO ou record. Um DTO representa dados de transporte; calcular taxa, determinar status ou transformar valores dentro dele viola SRP. Se aparecer um método de negócio dentro de um record que representa entrada ou saída da API, isso é um sinal de que a lógica foi parar no lugar errado.
- Criar um `PaymentDTO` no pacote `model` para carregar resultado intermediário. O lugar correto para um objeto que transporta dados entre as camadas internas do service é o pacote `service` (como o `ProcessedPayment`). O pacote `model` existe para representar documentos do banco.

## Resumo

Construímos uma Payment API em Java 17 com Spring Boot e MongoDB.

Primeiro criamos a versão `legacy-payment`, que funciona mas mistura responsabilidades dentro do mesmo service: validação, construção do objeto, cálculo de taxa, determinação de status e persistência convivem no mesmo método.

Depois criamos a versão `payments`, onde cada responsabilidade foi para uma classe específica:

- `ValidatePayment` decide o status
- `PaymentMapper` constrói o objeto `Payment`
- Cada `*PaymentProcessor` cuida do seu tipo
- `PaymentProcessorResolver` encontra o processador certo
- `PaymentService` só orquestra

Isso mostra SOLID na prática: não como teoria, mas como uma forma de deixar o código mais fácil de alterar. Se amanhã entrar PIX parcelado, só precisamos criar uma nova classe que implementa `PaymentProcessor`. Nenhuma outra classe muda.

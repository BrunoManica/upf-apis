# Tutorial: Criando um API Gateway com Spring Cloud Gateway

## Objetivo da aula

Vamos construir um exemplo pequeno de API Gateway usando Java 17, Spring Boot e Spring Cloud Gateway.

A ideia é criar três aplicações:

* `users-service`: uma API simples de usuários.
* `products-service`: uma API simples de produtos.
* `gateway-service`: o gateway que recebe as requisições e encaminha para os serviços.

Mesmo sendo um exemplo pequeno, vamos manter um padrão profissional:

* controller fino;
* service concentrando a regra;
* model separado;
* DTO separado;
* mapper manual separado;
* Lombok para reduzir código repetitivo;
* builder nos models;
* DTOs como `record` com `@Builder`.

O padrão dos mappers segue a mesma ideia usada no projeto de referência `fsj-receituario-api`: uma classe dedicada, método estático `mapFrom(...)`, null-check simples e conversão explícita entre um objeto e outro.

## Resultado final

No final, teremos três aplicações rodando:

```text
gateway-service  -> http://localhost:8080
users-service    -> http://localhost:8081
products-service -> http://localhost:8082
```

O cliente vai chamar o gateway:

```bash
curl http://localhost:8080/api/users
curl http://localhost:8080/api/products
```

E o gateway vai encaminhar internamente:

```text
/api/users    -> http://localhost:8081/users
/api/products -> http://localhost:8082/products
```

Também vamos deixar claro onde entram três recursos muito comuns em gateways reais:

* autenticação, para validar quem está chamando;
* rate limiting, para limitar excesso de chamadas;
* cache, para reaproveitar respostas seguras por um curto período.

## Contexto

Quando temos uma API só, o cliente chama essa API diretamente.

Quando temos vários microsserviços, o cliente poderia chamar cada um diretamente, mas isso começa a espalhar detalhes internos do backend.

Sem gateway, o cliente precisaria saber:

```text
http://localhost:8081/users
http://localhost:8082/products
```

Com gateway, o cliente conhece apenas:

```text
http://localhost:8080
```

O gateway recebe a chamada e decide para onde enviar.

## Explicação conceitual

O Spring Cloud Gateway trabalha com rotas.

Uma rota diz:

```text
quando chegar uma requisição com tal caminho,
encaminhe para tal serviço.
```

Por exemplo:

```text
/api/users -> users-service
```

O gateway não vai buscar usuário no banco.

Ele também não vai conter a regra de negócio de usuário.

Ele só vai receber a requisição e encaminhar para quem realmente sabe responder.

Esse detalhe é importante porque gateway não é lugar para concentrar toda a lógica da aplicação.

Além de rotear, um gateway pode aplicar regras transversais.

Regra transversal é uma regra que não pertence a um único serviço de negócio, mas aparece em várias chamadas do sistema.

Autenticação é um exemplo: antes de chegar no serviço de usuários ou no serviço de produtos, talvez a requisição precise provar quem está chamando.

Rate limiting é outro exemplo: antes de deixar um cliente chamar uma API mil vezes em poucos segundos, o gateway pode bloquear o excesso.

Cache também pode entrar nessa camada: se uma resposta pública muda pouco, o gateway pode guardar essa resposta por alguns segundos e evitar chamadas repetidas ao serviço interno.

O cuidado é não transformar o gateway em um serviço cheio de regra de negócio.

Ele pode controlar a entrada.

Quem decide regra de usuário, produto, pedido ou pagamento continua sendo o serviço responsável por aquele domínio.

## Setup inicial

Você precisa ter instalado:

* Java 17.
* Um terminal.
* Um editor de código.

Vamos usar o Gradle Wrapper gerado pelo Spring Initializr.

Isso evita depender de uma instalação global do Gradle na máquina.

Crie uma pasta para a aula:

```bash
mkdir gateway-demo
cd gateway-demo
```

Se estiver no Windows, você pode criar a pasta pelo Explorer e abrir o terminal dentro dela.

## Passo a passo

### 1. Criar o users-service

Comece criando uma aplicação Spring Boot para usuários.

No Spring Initializr, use:

* Project: Gradle.
* Language: Java.
* Spring Boot: versão estável 3.x.
* Java: 17.
* Group: `br.edu.upf`
* Artifact: `users-service`
* Dependencies: Spring Web e Lombok.

No `build.gradle`, o bloco de dependências precisa ter o Spring Web e o Lombok:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

O Lombok reduz código repetitivo. Nesta aula, vamos usá-lo principalmente para `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor` e `@RequiredArgsConstructor`.

Agora crie estes pacotes:

```text
users-service/src/main/java/br/edu/upf/usersservice/
├── controller/
├── dto/
├── mapper/
├── model/
└── service/
```

Cada coisa fica no seu lugar:

* `controller`: recebe HTTP.
* `service`: concentra a regra.
* `model`: representa o dado interno da aplicação.
* `dto`: representa o dado que sai pela API.
* `mapper`: converte model para DTO.

### 2. Criar o model de usuário

Crie o arquivo:

```text
users-service/src/main/java/br/edu/upf/usersservice/model/User.java
```

Com este conteúdo:

```java
package br.edu.upf.usersservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    private Long id;
    private String name;
    private String email;
}
```

O `User` é o model interno do serviço.

Como ainda não temos banco de dados nesta aula, ele não tem anotação de persistência.

O `@Builder` permite criar objetos de forma legível, sem depender de construtores longos.

### 3. Criar o DTO de resposta de usuário

Crie o arquivo:

```text
users-service/src/main/java/br/edu/upf/usersservice/dto/UserResponse.java
```

Com este conteúdo:

```java
package br.edu.upf.usersservice.dto;

import lombok.Builder;

@Builder
public record UserResponse(Long id, String name, String email) {
}
```

O DTO representa o que a API devolve para fora.

Ele fica separado do model porque o formato interno da aplicação não precisa ser obrigatoriamente igual ao formato público da API.

Aqui usamos `record` porque DTO de resposta é apenas um pacote de dados para sair pela API.

Também usamos `@Builder` para manter a criação do DTO no mapper mais legível.

### 4. Criar o mapper de usuário

Crie o arquivo:

```text
users-service/src/main/java/br/edu/upf/usersservice/mapper/UserResponseMapper.java
```

Com este conteúdo:

```java
package br.edu.upf.usersservice.mapper;

import br.edu.upf.usersservice.dto.UserResponse;
import br.edu.upf.usersservice.model.User;

import java.util.List;

public final class UserResponseMapper {

    private UserResponseMapper() {
    }

    public static UserResponse mapFrom(User user) {
        if (user == null)
            return null;

        return UserResponse.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .build();
    }

    public static List<UserResponse> mapFrom(List<User> users) {
        if (users == null)
            return List.of();

        return users.stream()
                .map(UserResponseMapper::mapFrom)
                .toList();
    }
}
```

Esse mapper segue um padrão simples:

* classe dedicada;
* construtor privado, porque ninguém precisa instanciar o mapper;
* método `mapFrom(...)`;
* null-check antes de converter;
* criação do DTO usando `builder()`, mesmo o DTO sendo um `record`.

O controller não deve fazer essa conversão.

### 5. Criar o service de usuários

Crie o arquivo:

```text
users-service/src/main/java/br/edu/upf/usersservice/service/UserService.java
```

Com este conteúdo:

```java
package br.edu.upf.usersservice.service;

import br.edu.upf.usersservice.dto.UserResponse;
import br.edu.upf.usersservice.mapper.UserResponseMapper;
import br.edu.upf.usersservice.model.User;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class UserService {

    public List<UserResponse> findAll() {
        List<User> users = List.of(
                User.builder()
                        .id(1L)
                        .name("Ana")
                        .email("ana@email.com")
                        .build(),
                User.builder()
                        .id(2L)
                        .name("Lucas")
                        .email("lucas@email.com")
                        .build(),
                User.builder()
                        .id(3L)
                        .name("Maria")
                        .email("maria@email.com")
                        .build()
        );

        return UserResponseMapper.mapFrom(users);
    }
}
```

O service concentra a lógica da aplicação.

Por enquanto, a lista está fixa no código para a aula ficar leve.

Em uma aplicação real, esse service chamaria um repository para buscar os dados no banco.

O importante aqui é perceber que o controller não monta a lista e não cria DTO.

### 6. Criar o controller de usuários

Crie o arquivo:

```text
users-service/src/main/java/br/edu/upf/usersservice/controller/UserController.java
```

Com este conteúdo:

```java
package br.edu.upf.usersservice.controller;

import br.edu.upf.usersservice.dto.UserResponse;
import br.edu.upf.usersservice.service.UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public List<UserResponse> findAll() {
        return userService.findAll();
    }
}
```

Agora o controller ficou fino.

Ele só recebe a requisição HTTP e chama o service.

Não tem lista fixa no controller.

Não tem regra de negócio no controller.

Não tem mapper no controller.

### 7. Configurar e testar o users-service

Crie ou edite:

```text
users-service/src/main/resources/application.properties
```

Com este conteúdo:

```properties
spring.application.name=users-service
server.port=8081
```

Rode a aplicação:

```bash
cd /caminho/para/gateway-demo/users-service
./gradlew bootRun
```

No Windows, use:

```bash
cd /caminho/para/gateway-demo/users-service
gradlew.bat bootRun
```

Teste em outro terminal:

```bash
curl http://localhost:8081/users
```

A resposta esperada é:

```json
[
  {
    "id": 1,
    "name": "Ana",
    "email": "ana@email.com"
  },
  {
    "id": 2,
    "name": "Lucas",
    "email": "lucas@email.com"
  },
  {
    "id": 3,
    "name": "Maria",
    "email": "maria@email.com"
  }
]
```

Nesse momento ainda não existe gateway.

Estamos apenas garantindo que o primeiro serviço responde sozinho.

### 8. Criar o products-service

Agora crie outra aplicação Spring Boot para produtos.

No Spring Initializr, use:

* Project: Gradle.
* Language: Java.
* Spring Boot: versão estável 3.x.
* Java: 17.
* Group: `br.edu.upf`
* Artifact: `products-service`
* Dependencies: Spring Web e Lombok.

No `build.gradle`, use a mesma base de dependências do `users-service`:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

Crie estes pacotes:

```text
products-service/src/main/java/br/edu/upf/productsservice/
├── controller/
├── dto/
├── mapper/
├── model/
└── service/
```

### 9. Criar o model de produto

Crie o arquivo:

```text
products-service/src/main/java/br/edu/upf/productsservice/model/Product.java
```

Com este conteúdo:

```java
package br.edu.upf.productsservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    private Long id;
    private String name;
    private BigDecimal price;
}
```

O `Product` é o model interno do serviço de produtos.

Usamos `BigDecimal` para preço porque ele representa valores monetários melhor do que `double`.

### 10. Criar o DTO de resposta de produto

Crie o arquivo:

```text
products-service/src/main/java/br/edu/upf/productsservice/dto/ProductResponse.java
```

Com este conteúdo:

```java
package br.edu.upf.productsservice.dto;

import lombok.Builder;

import java.math.BigDecimal;

@Builder
public record ProductResponse(Long id, String name, BigDecimal price) {
}
```

O DTO fica separado pelo mesmo motivo do serviço de usuários: ele representa a resposta pública da API.

### 11. Criar o mapper de produto

Crie o arquivo:

```text
products-service/src/main/java/br/edu/upf/productsservice/mapper/ProductResponseMapper.java
```

Com este conteúdo:

```java
package br.edu.upf.productsservice.mapper;

import br.edu.upf.productsservice.dto.ProductResponse;
import br.edu.upf.productsservice.model.Product;

import java.util.List;

public final class ProductResponseMapper {

    private ProductResponseMapper() {
    }

    public static ProductResponse mapFrom(Product product) {
        if (product == null)
            return null;

        return ProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .price(product.getPrice())
                .build();
    }

    public static List<ProductResponse> mapFrom(List<Product> products) {
        if (products == null)
            return List.of();

        return products.stream()
                .map(ProductResponseMapper::mapFrom)
                .toList();
    }
}
```

Esse mapper é praticamente igual ao mapper de usuários.

Essa repetição é boa para o começo, porque o aluno enxerga o mesmo padrão em outro domínio.

### 12. Criar o service de produtos

Crie o arquivo:

```text
products-service/src/main/java/br/edu/upf/productsservice/service/ProductService.java
```

Com este conteúdo:

```java
package br.edu.upf.productsservice.service;

import br.edu.upf.productsservice.dto.ProductResponse;
import br.edu.upf.productsservice.mapper.ProductResponseMapper;
import br.edu.upf.productsservice.model.Product;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.List;

@Service
public class ProductService {

    public List<ProductResponse> findAll() {
        List<Product> products = List.of(
                Product.builder()
                        .id(1L)
                        .name("Notebook")
                        .price(new BigDecimal("3500.00"))
                        .build(),
                Product.builder()
                        .id(2L)
                        .name("Mouse")
                        .price(new BigDecimal("80.00"))
                        .build(),
                Product.builder()
                        .id(3L)
                        .name("Teclado")
                        .price(new BigDecimal("180.00"))
                        .build()
        );

        return ProductResponseMapper.mapFrom(products);
    }
}
```

De novo, a lista fixa está no service apenas para manter o exemplo pequeno.

O controller continua sem regra.

### 13. Criar o controller de produtos

Crie o arquivo:

```text
products-service/src/main/java/br/edu/upf/productsservice/controller/ProductController.java
```

Com este conteúdo:

```java
package br.edu.upf.productsservice.controller;

import br.edu.upf.productsservice.dto.ProductResponse;
import br.edu.upf.productsservice.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public List<ProductResponse> findAll() {
        return productService.findAll();
    }
}
```

O controller de produtos segue o mesmo padrão do controller de usuários.

### 14. Configurar e testar o products-service

Crie ou edite:

```text
products-service/src/main/resources/application.properties
```

Com este conteúdo:

```properties
spring.application.name=products-service
server.port=8082
```

Rode a aplicação:

```bash
cd /caminho/para/gateway-demo/products-service
./gradlew bootRun
```

No Windows:

```bash
cd /caminho/para/gateway-demo/products-service
gradlew.bat bootRun
```

Teste:

```bash
curl http://localhost:8082/products
```

A resposta esperada é:

```json
[
  {
    "id": 1,
    "name": "Notebook",
    "price": 3500.00
  },
  {
    "id": 2,
    "name": "Mouse",
    "price": 80.00
  },
  {
    "id": 3,
    "name": "Teclado",
    "price": 180.00
  }
]
```

Agora temos dois serviços independentes.

Cada um responde em uma porta diferente.

### 15. Criar o gateway-service

Agora faz sentido criar o gateway.

Crie mais uma aplicação Spring Boot.

No Spring Initializr, use:

* Project: Gradle.
* Language: Java.
* Spring Boot: versão estável 3.x.
* Java: 17.
* Group: `br.edu.upf`
* Artifact: `gateway-service`
* Dependencies: Gateway.

No Spring Initializr, a dependência pode aparecer como:

```text
Gateway
```

ou:

```text
Spring Cloud Gateway
```

Essa é a dependência que traz o Spring Cloud Gateway para o projeto.

O gateway não precisa de `Spring Web` neste exemplo.

Spring Cloud Gateway usa uma base reativa por baixo dos panos. Isso significa que ele trabalha com outro motor HTTP do Spring, chamado WebFlux. Para esta aula, não precisamos aprofundar nisso. O ponto importante é: no gateway, usamos a dependência de Gateway, não criamos controller para cada rota.

Depois de gerar o projeto, confira o `build.gradle` do `gateway-service`.

Na versão atual do Spring Cloud, o starter usado neste tutorial é o WebFlux:

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway-server-webflux'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

Se o Spring Initializr gerar esta dependência:

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc'
```

troque pela versão WebFlux mostrada acima.

Esse ajuste é importante porque a configuração de rotas que vamos usar no `application.yml` fica em:

```text
spring.cloud.gateway.server.webflux.routes
```

### 16. Configurar as rotas do gateway

Para configurar rotas com mais clareza, vamos usar YAML.

Renomeie o arquivo:

```text
gateway-service/src/main/resources/application.properties
```

Para:

```text
gateway-service/src/main/resources/application.yml
```

Agora coloque este conteúdo:

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: users-route
              uri: http://localhost:8081
              predicates:
                - Path=/api/users/**
              filters:
                - RewritePath=/api/users(?<segment>/?.*), /users$\{segment}

            - id: products-route
              uri: http://localhost:8082
              predicates:
                - Path=/api/products/**
              filters:
                - RewritePath=/api/products(?<segment>/?.*), /products$\{segment}

server:
  port: 8080
```

Esta rota:

```yaml
- id: users-route
  uri: http://localhost:8081
  predicates:
    - Path=/api/users/**
```

diz que toda requisição que chegar em `/api/users/**` deve ser encaminhada para `http://localhost:8081`.

O filtro:

```yaml
- RewritePath=/api/users(?<segment>/?.*), /users$\{segment}
```

remove o prefixo `/api` antes de enviar a requisição para o serviço.

Então o caminho fica assim:

```text
Cliente chama:
GET /api/users

Gateway encaminha para:
GET /users
```

O mesmo raciocínio vale para produtos.

```text
Cliente chama:
GET /api/products

Gateway encaminha para:
GET /products
```

Esse é o coração da aula.

### 17. Rodar os três projetos

Agora precisamos deixar as três aplicações rodando ao mesmo tempo.

Use três terminais.

Terminal 1:

```bash
cd /caminho/para/gateway-demo/users-service
./gradlew bootRun
```

Terminal 2:

```bash
cd /caminho/para/gateway-demo/products-service
./gradlew bootRun
```

Terminal 3:

```bash
cd /caminho/para/gateway-demo/gateway-service
./gradlew bootRun
```

No Windows, troque `./gradlew` por:

```bash
gradlew.bat
```

### 18. Testar os serviços diretamente

Antes de testar o gateway, teste os serviços diretamente.

Usuários:

```bash
curl http://localhost:8081/users
```

Produtos:

```bash
curl http://localhost:8082/products
```

Isso confirma que os serviços estão funcionando.

Se um serviço não responde diretamente, o gateway também não vai conseguir encaminhar para ele.

### 19. Testar pelo gateway

Agora teste pelo gateway.

Usuários:

```bash
curl http://localhost:8080/api/users
```

Produtos:

```bash
curl http://localhost:8080/api/products
```

Se tudo estiver certo, as respostas serão iguais às respostas diretas dos serviços.

A diferença é o caminho que a requisição percorreu.

Antes:

```text
Cliente -> users-service
```

Agora:

```text
Cliente -> gateway-service -> users-service
```

### 20. Entender o fluxo completo

Quando você executa:

```bash
curl http://localhost:8080/api/users
```

acontece isto:

```text
1. A requisição chega no gateway-service pela porta 8080.
2. O Spring Cloud Gateway procura uma rota compatível.
3. Ele encontra a rota com Path=/api/users/**.
4. O filtro RewritePath transforma /api/users em /users.
5. O gateway encaminha a requisição para http://localhost:8081/users.
6. O users-service passa por controller, service e mapper.
7. O gateway devolve essa resposta para o cliente.
```

O cliente não precisou saber que o serviço de usuários roda na porta `8081`.

Ele só conhece o gateway.

### 21. Adicionar um header para enxergar que passou pelo gateway

Agora vamos adicionar um detalhe simples para visualizar melhor o gateway trabalhando.

No `application.yml` do `gateway-service`, adicione um filtro `AddResponseHeader` em cada rota:

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: users-route
              uri: http://localhost:8081
              predicates:
                - Path=/api/users/**
              filters:
                - RewritePath=/api/users(?<segment>/?.*), /users$\{segment}
                - AddResponseHeader=X-Gateway, gateway-service

            - id: products-route
              uri: http://localhost:8082
              predicates:
                - Path=/api/products/**
              filters:
                - RewritePath=/api/products(?<segment>/?.*), /products$\{segment}
                - AddResponseHeader=X-Gateway, gateway-service

server:
  port: 8080
```

Esse filtro adiciona um header HTTP na resposta.

Rode:

```bash
curl -i http://localhost:8080/api/users
```

Procure por este header:

```text
X-Gateway: gateway-service
```

Isso mostra que a resposta passou pelo gateway antes de voltar para o cliente.

### 22. Entender autenticação no gateway

Agora que o roteamento está funcionando, faz sentido falar de autenticação.

Autenticação responde uma pergunta simples:

```text
Quem está fazendo esta chamada?
```

Em APIs REST, isso costuma aparecer com um token no header:

```text
Authorization: Bearer token-aqui
```

O gateway pode validar esse token antes de encaminhar a requisição.

Se o token for válido, a chamada segue.

Se o token estiver ausente ou inválido, o gateway responde `401 Unauthorized`.

O caminho fica assim:

```text
Cliente -> Gateway valida token -> Serviço interno
```

Isso é útil porque evita repetir a mesma verificação básica em todos os serviços.

Em Spring, essa evolução normalmente usa Spring Security com OAuth2 Resource Server no `gateway-service`.

O `build.gradle` do gateway receberia uma dependência como esta:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
```

Depois, o gateway teria uma configuração de segurança parecida com esta:

```java
package br.edu.upf.gatewayservice.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
                .csrf(ServerHttpSecurity.CsrfSpec::disable)
                .authorizeExchange(exchanges -> exchanges
                        .pathMatchers("/api/products/**").permitAll()
                        .pathMatchers("/api/users/**").authenticated()
                        .anyExchange().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                .build();
    }
}
```

Não vamos ativar isso no exemplo principal porque exigiria um emissor real de tokens, como Keycloak, Auth0, Cognito ou outro servidor de identidade.

O importante agora é entender o lugar dessa responsabilidade: o gateway é uma boa camada para validar a chamada antes de ela chegar aos serviços internos.

Mesmo assim, em produção, os serviços internos não devem ficar expostos livremente para a internet.

### 23. Entender rate limiting no gateway

Rate limiting limita quantas requisições um cliente pode fazer em um intervalo de tempo.

Um exemplo simples:

```text
até 10 requisições por segundo
com pico máximo de 20 requisições acumuladas
```

Quando o cliente passa do limite, o gateway responde:

```text
429 Too Many Requests
```

Esse recurso protege o sistema contra abuso, erro de cliente, script mal configurado e picos que poderiam derrubar os serviços internos.

No Spring Cloud Gateway, uma forma comum de fazer isso é com o filtro `RequestRateLimiter`.

Para funcionar bem em mais de uma instância do gateway, normalmente usamos Redis, porque o limite precisa ser compartilhado entre as instâncias.

O `build.gradle` do gateway receberia:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
```

O `application.yml` poderia evoluir assim:

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: users-route
              uri: http://localhost:8081
              predicates:
                - Path=/api/users/**
              filters:
                - RewritePath=/api/users(?<segment>/?.*), /users$\{segment}
                - AddResponseHeader=X-Gateway, gateway-service
                - name: RequestRateLimiter
                  args:
                    redis-rate-limiter.replenishRate: 10
                    redis-rate-limiter.burstCapacity: 20
                    redis-rate-limiter.requestedTokens: 1
                    key-resolver: "#{@ipKeyResolver}"
```

O `replenishRate` indica quantos tokens voltam para o limite a cada segundo.

O `burstCapacity` indica o pico máximo permitido.

O `key-resolver` define como o gateway identifica quem está consumindo o limite.

Para um exemplo simples, poderia ser pelo IP:

```java
package br.edu.upf.gatewayservice.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

import java.net.InetSocketAddress;

@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();

            if (remoteAddress == null || remoteAddress.getAddress() == null)
                return Mono.just("unknown");

            String hostAddress = remoteAddress.getAddress().getHostAddress();

            return Mono.just(hostAddress);
        };
    }
}
```

Em sistemas reais, limitar apenas por IP nem sempre é suficiente.

Quando existe autenticação, é comum limitar por usuário, cliente, aplicação parceira ou plano contratado.

### 24. Entender cache no gateway

Cache no gateway serve para reaproveitar uma resposta por um tempo curto.

Ele faz sentido principalmente em chamadas `GET` que retornam dados públicos ou pouco variáveis.

Um exemplo bom para começar seria:

```text
GET /api/products
```

Se a lista de produtos muda pouco, o gateway pode guardar a resposta por alguns segundos.

A primeira chamada vai até o serviço:

```text
Cliente -> Gateway -> products-service
```

As próximas chamadas, enquanto o cache ainda está válido, podem voltar direto do gateway:

```text
Cliente -> Gateway -> cache
```

Isso melhora o tempo de resposta e reduz carga no serviço de produtos.

No Spring Cloud Gateway, a ideia pode aparecer com cache local de resposta.

Para isso, o gateway também precisa das dependências de cache:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'com.github.ben-manes.caffeine:caffeine'
```

Uma configuração conceitual ficaria assim:

```yaml
spring:
  cloud:
    gateway:
      filter:
        local-response-cache:
          enabled: true
      server:
        webflux:
          routes:
            - id: products-route
              uri: http://localhost:8082
              predicates:
                - Path=/api/products/**
              filters:
                - RewritePath=/api/products(?<segment>/?.*), /products$\{segment}
                - LocalResponseCache=30s,10MB
```

Aqui, a resposta pode ser reaproveitada por até `30s`, respeitando um espaço máximo de cache.

Não use cache em qualquer rota.

Evite cachear:

* dados privados de usuário;
* tokens;
* pedidos em andamento;
* carrinho de compras;
* pagamento;
* saldo;
* qualquer resposta que muda conforme quem está autenticado.

Cache é ótimo quando a resposta pode ser compartilhada com segurança.

Quando a resposta depende da identidade do usuário, pare e pense antes de colocar cache no gateway.

## Código completo

### users-service

Arquivos principais:

```text
users-service/src/main/java/br/edu/upf/usersservice/
├── UsersServiceApplication.java
├── controller/UserController.java
├── dto/UserResponse.java
├── mapper/UserResponseMapper.java
├── model/User.java
└── service/UserService.java
```

O código completo desses arquivos é o mesmo que foi criado no passo a passo.

### products-service

Arquivos principais:

```text
products-service/src/main/java/br/edu/upf/productsservice/
├── ProductsServiceApplication.java
├── controller/ProductController.java
├── dto/ProductResponse.java
├── mapper/ProductResponseMapper.java
├── model/Product.java
└── service/ProductService.java
```

O código completo desses arquivos é o mesmo que foi criado no passo a passo.

### gateway-service

Arquivo:

```text
gateway-service/src/main/resources/application.yml
```

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: users-route
              uri: http://localhost:8081
              predicates:
                - Path=/api/users/**
              filters:
                - RewritePath=/api/users(?<segment>/?.*), /users$\{segment}
                - AddResponseHeader=X-Gateway, gateway-service

            - id: products-route
              uri: http://localhost:8082
              predicates:
                - Path=/api/products/**
              filters:
                - RewritePath=/api/products(?<segment>/?.*), /products$\{segment}
                - AddResponseHeader=X-Gateway, gateway-service

server:
  port: 8080
```

Arquivo:

```text
gateway-service/src/main/java/br/edu/upf/gatewayservice/GatewayServiceApplication.java
```

```java
package br.edu.upf.gatewayservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayServiceApplication.class, args);
    }
}
```

Perceba que o gateway não tem controller.

Nesse exemplo, quem faz o roteamento é a configuração do Spring Cloud Gateway.

Se o gateway mostrar um aviso dizendo que `spring.cloud.gateway.routes` foi renomeado, use a estrutura atual com `spring.cloud.gateway.server.webflux.routes`, igual ao arquivo acima.

## Erros comuns

### Colocar tudo dentro do controller

Evite montar lista, criar model, converter DTO ou aplicar regra de negócio dentro do controller.

O controller recebe HTTP e chama o service.

Essa separação parece mais trabalhosa no começo, mas evita que a aplicação vire um arquivo grande e confuso.

### Declarar DTO dentro do controller

O DTO deve ficar em arquivo próprio dentro do pacote `dto`.

Declarar DTO dentro do controller parece prático, mas mistura responsabilidades e dificulta a evolução do projeto.

### Colocar mapper dentro do controller

O controller não deve chamar mapper.

Neste exemplo, o service retorna o DTO já pronto porque a montagem da resposta faz parte da orquestração da aplicação, não da camada HTTP.

### Esquecer o Lombok no build.gradle

Se o projeto não compilar reconhecendo `@Builder`, `@Data` ou `@RequiredArgsConstructor`, confira se o Lombok está no `build.gradle` como `compileOnly` e `annotationProcessor`.

### O gateway sobe, mas a rota retorna erro

Verifique se o serviço de destino está rodando.

Se você chamar:

```bash
curl http://localhost:8080/api/users
```

e o `users-service` estiver parado, o gateway não tem para onde encaminhar.

Primeiro teste:

```bash
curl http://localhost:8081/users
```

### A rota retorna 404

Confira o caminho configurado no `Path`.

Se a rota está assim:

```yaml
- Path=/api/users/**
```

então a chamada precisa começar com:

```text
/api/users
```

Também confira o `RewritePath`.

Se o serviço espera `/users`, o gateway precisa encaminhar para `/users`, não para `/api/users`.

### Usar a mesma porta em dois projetos

Cada aplicação precisa de uma porta diferente.

Neste tutorial usamos:

```text
8080 -> gateway-service
8081 -> users-service
8082 -> products-service
```

Se duas aplicações tentarem subir na mesma porta, uma delas vai falhar.

## Resumo

Construímos três aplicações Spring Boot:

* `users-service`, que responde `GET /users` usando controller, service, model, DTO e mapper manual.
* `products-service`, que responde `GET /products` usando a mesma separação.
* `gateway-service`, que recebe `/api/users` e `/api/products`.

O ponto principal da aula é:

```text
Cliente chama o gateway.
Gateway decide a rota.
Serviço responde.
Gateway devolve a resposta.
```

E o ponto principal de organização de código é:

```text
Controller recebe HTTP.
Service concentra a regra.
Mapper converte model para DTO.
DTO representa a resposta da API.
Model representa o dado interno.
```

Essa é a base para evoluir depois com autenticação, rate limiting, logs, métricas, Docker, banco de dados e outros recursos.

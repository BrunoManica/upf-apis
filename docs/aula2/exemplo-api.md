# Aula 2 - Criando uma API REST com Spring Boot e MongoDB

## Objetivo da aula

Nesta aula vamos construir uma API REST simples usando Java 17, Spring Boot e MongoDB.

A ideia é sair da teoria sobre APIs e criar um CRUD de usuários com as operações mais comuns:

* criar usuário;
* listar usuários;
* buscar usuário por id;
* atualizar usuário;
* remover usuário.

No final, você terá uma API funcionando localmente, com banco MongoDB em Docker e documentação Swagger para testar os endpoints pelo navegador.

## Resultado final

Ao terminar a aula, a aplicação terá estes endpoints:

```text
POST   /users
GET    /users
GET    /users/{id}
PUT    /users/{id}
DELETE /users/{id}
```

A estrutura principal ficará assim:

```text
minha-api/
+-- src/
|   +-- main/
|       +-- java/
|       |   +-- br/edu/upf/minha_api/
|       |       +-- MinhaApiApplication.java
|       |       +-- config/
|       |       |   +-- OpenApiConfig.java
|       |       +-- users/
|       |           +-- User.java
|       |           +-- UserRepository.java
|       |           +-- UserService.java
|       |           +-- UserController.java
|       |           +-- CreateUserRequest.java
|       |           +-- UpdateUserRequest.java
|       +-- resources/
|           +-- application.properties
+-- build.gradle
+-- settings.gradle
+-- gradlew
+-- gradle/
+-- Dockerfile
+-- .dockerignore
+-- docker-compose.yml
```

Esse é o Spring Boot em uma estrutura simples por recurso. Como o exemplo é pequeno, os arquivos do CRUD de usuários ficam juntos no pacote `users`.

## Contexto

Uma API REST normalmente fica entre quem consome os dados e onde esses dados são armazenados.

Neste exemplo, o cliente pode ser um front-end, um aplicativo mobile, o Swagger, o Postman ou outro serviço. A API recebe a requisição HTTP, valida os dados, executa a regra necessária e salva ou consulta informações no MongoDB.

O fluxo principal será este:

```text
Cliente HTTP
  chama a API
Controller
  recebe a requisição
Service
  executa a regra de negócio
Repository
  conversa com o MongoDB
MongoDB
  armazena os documentos
```

Essa separação ajuda porque cada camada tem um papel claro. O controller não precisa saber como o MongoDB funciona. O repository não precisa conhecer regras de negócio. O service fica no meio coordenando o trabalho.

## Explicação conceitual

Antes de começar o código, vale entender as peças que vão aparecer.

`Controller` é a camada que recebe chamadas HTTP. Quando alguém faz um `POST`, `GET`, `PUT` ou `DELETE`, é o controller que atende primeiro.

`Service` é a camada onde colocamos a regra da aplicação. Neste exemplo, ela vai impedir email duplicado, buscar usuário por id e decidir o que acontece na atualização.

`Model` representa o dado salvo no MongoDB. Como estamos usando MongoDB, o model será um documento.

`Repository` é a camada que acessa o banco. O Spring Data MongoDB cria boa parte da implementação automaticamente.

`DTO` é o objeto usado para representar dados de entrada ou saída da API. Neste exemplo, vamos usar DTOs para entrada e devolver o `User` diretamente para manter a aula simples.

## Setup inicial

Você precisa ter instalado:

* Java JDK 17;
* Docker Desktop ou Docker Engine;
* um editor de código, como IntelliJ IDEA, VS Code ou Eclipse;
* navegador web;
* Postman ou Insomnia, se quiser testar fora do Swagger.

Links úteis:

* Java JDK 17: [https://adoptium.net/temurin/releases/?version=17](https://adoptium.net/temurin/releases/?version=17)
* Docker: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
* Spring Initializr: [https://start.spring.io/](https://start.spring.io/)
* Postman: [https://www.postman.com/downloads/](https://www.postman.com/downloads/)
* Insomnia: [https://insomnia.rest/download](https://insomnia.rest/download)

O Gradle será gerado junto com o projeto pelo Spring Initializr. Por isso, você não precisa instalar Gradle manualmente. Vamos usar o Gradle Wrapper, que vem nos arquivos `gradlew` e `gradlew.bat`.

## Passo a passo

### 1. Subir o MongoDB com Docker

Vamos começar pelo banco porque a API vai precisar dele para salvar usuários.

Execute:

```bash
docker run --name mongo-api \
  -p 27017:27017 \
  -v mongo_api_data:/data/db \
  -d mongo:7
```

Esse comando cria um container chamado `mongo-api`, expõe a porta `27017` e usa um volume chamado `mongo_api_data` para manter os dados mesmo se o container for parado.

Verifique se o container está rodando:

```bash
docker ps
```

Comandos úteis:

```bash
# Parar o MongoDB
docker stop mongo-api

# Iniciar novamente
docker start mongo-api

# Ver logs do container
docker logs mongo-api
```

### 2. Criar o projeto no Spring Initializr

Abra [https://start.spring.io/](https://start.spring.io/) no navegador.

Configure o projeto assim:

```text
Project: Gradle - Groovy
Language: Java
Spring Boot: versão estável sugerida pelo site
Group: br.edu.upf
Artifact: minha-api
Name: minha-api
Description: API REST com Spring Boot e MongoDB
Package name: br.edu.upf.minha_api
Packaging: Jar
Java: 17
```

Para evitar diferença entre o que o Spring Initializr gera e o que vamos usar no código, adicione somente:

* Spring Web.

Depois clique em `Generate` para baixar o `.zip`.

Depois de abrir o projeto, vamos sobrescrever o `build.gradle` inteiro. Isso garante que MongoDB, validação, Swagger e Lombok fiquem exatamente como o projeto precisa.

### 3. Descompactar e abrir o projeto

No terminal, vá para a pasta onde o arquivo foi baixado e execute:

```bash
unzip minha-api.zip -d minha-api
cd minha-api
```

Abra a pasta `minha-api` no seu editor.

### 4. Configurar a aplicação

Edite `src/main/resources/application.properties`:

```properties
spring.application.name=minha-api
server.port=${PORT:8080}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/minha_api}

springdoc.swagger-ui.path=/swagger-ui.html
```

Essa configuração faz três coisas importantes.

`server.port=${PORT:8080}` usa a variável `PORT` quando ela existir. Se não existir, a API sobe na porta `8080`.

`spring.data.mongodb.uri` aponta para o MongoDB local por padrão. Quando a aplicação rodar em container, vamos sobrescrever esse valor com `MONGODB_URI`.

`springdoc.swagger-ui.path` define o endereço da tela do Swagger.

### 5. Substituir o build.gradle

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

Esse arquivo usa o starter `spring-boot-starter-webmvc`, que é o nome gerado nas versões novas do Spring Boot. Se o editor pedir para recarregar o projeto Gradle, aceite.

### 6. Configurar informações do Swagger

Crie a pasta:

```text
src/main/java/br/edu/upf/minha_api/config
```

Crie o arquivo `src/main/java/br/edu/upf/minha_api/config/OpenApiConfig.java`:

```java
package br.edu.upf.minha_api.config;

import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import org.springframework.context.annotation.Configuration;

@Configuration
@OpenAPIDefinition(
  info = @Info(
    title = "Minha API",
    version = "1.0",
    description = "API de usuarios com Spring Boot e MongoDB"
  )
)
public class OpenApiConfig {
}
```

`@Configuration` diz ao Spring que essa classe participa da configuração da aplicação.

`@OpenAPIDefinition` personaliza as informações que aparecem no Swagger.

### 7. Criar o model

Crie a pasta:

```text
src/main/java/br/edu/upf/minha_api/users
```

Crie o arquivo `src/main/java/br/edu/upf/minha_api/users/User.java`:

```java
package br.edu.upf.minha_api.users;

import java.time.Instant;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@NoArgsConstructor
@Document(collection = "users")
public class User {
  @Id
  private String id;

  private String name;

  @Indexed(unique = true)
  private String email;

  private Instant createdAt = Instant.now();
}
```

`@Document(collection = "users")` informa que essa classe será salva na coleção `users` do MongoDB.

`@Id` marca o campo usado como identificador do documento.

`@Indexed(unique = true)` pede ao MongoDB um índice único para o email. Isso ajuda a evitar dois usuários com o mesmo email.

`@Data` e `@NoArgsConstructor` vêm do Lombok. Eles geram getters, setters e construtor vazio, evitando escrever código repetitivo nessa aula.

### 8. Criar os DTOs

Os DTOs de entrada também ficam no pacote `users`, junto com os outros arquivos desse recurso.

Crie o arquivo `src/main/java/br/edu/upf/minha_api/users/CreateUserRequest.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record CreateUserRequest(
  @NotBlank(message = "Nome e obrigatorio")
  String name,

  @NotBlank(message = "Email e obrigatorio")
  @Email(message = "Email invalido")
  String email
) {}
```

Esse DTO representa o corpo da requisição de criação.

`@NotBlank` impede valores vazios.

`@Email` valida se o texto tem formato de email.

Agora crie `src/main/java/br/edu/upf/minha_api/users/UpdateUserRequest.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.constraints.Email;

public record UpdateUserRequest(
  String name,

  @Email(message = "Email invalido")
  String email
) {}
```

Na atualização, os campos podem vir vazios ou nulos porque vamos permitir atualizar apenas parte dos dados.

### 9. Criar o repository

Crie o arquivo `src/main/java/br/edu/upf/minha_api/users/UserRepository.java`:

```java
package br.edu.upf.minha_api.users;

import java.util.Optional;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface UserRepository extends MongoRepository<User, String> {
  Optional<User> findByEmail(String email);
}
```

`MongoRepository<User, String>` diz duas coisas:

* o repository trabalha com a classe `User`;
* o tipo do id é `String`.

O método `findByEmail` é criado automaticamente pelo Spring Data MongoDB a partir do nome do método.

### 10. Criar o service

Crie o arquivo `src/main/java/br/edu/upf/minha_api/users/UserService.java`:

```java
package br.edu.upf.minha_api.users;

import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class UserService {
  private final UserRepository repository;

  public UserService(UserRepository repository) {
    this.repository = repository;
  }

  public User create(CreateUserRequest request) {
    var existingUser = repository.findByEmail(request.email());

    if (existingUser.isPresent()) {
      throw new ResponseStatusException(HttpStatus.CONFLICT, "Email ja cadastrado");
    }

    var user = new User();
    user.setName(request.name());
    user.setEmail(request.email());

    return repository.save(user);
  }

  public List<User> findAll() {
    return repository.findAll();
  }

  public User findOne(String id) {
    return repository.findById(id)
      .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Usuario nao encontrado"));
  }

  public User update(String id, UpdateUserRequest request) {
    var user = findOne(id);

    updateName(user, request.name());
    updateEmail(user, request.email());

    return repository.save(user);
  }

  public void remove(String id) {
    var user = findOne(id);
    repository.delete(user);
  }

  private void updateName(User user, String name) {
    if (name == null || name.isBlank()) {
      return;
    }

    user.setName(name);
  }

  private void updateEmail(User user, String email) {
    if (email == null || email.isBlank()) {
      return;
    }

    var existingUser = repository.findByEmail(email);

    if (existingUser.isEmpty()) {
      user.setEmail(email);
      return;
    }

    if (existingUser.get().getId().equals(user.getId())) {
      user.setEmail(email);
      return;
    }

    throw new ResponseStatusException(HttpStatus.CONFLICT, "Email ja cadastrado");
  }
}
```

O service concentra as decisões da aplicação.

Na criação, ele verifica se já existe usuário com o mesmo email. Se existir, devolve `409 Conflict`.

Na busca, atualização e remoção, ele usa `findOne`. Assim a regra de "usuário não encontrado" fica em um único lugar.

### 11. Criar o controller

Crie o arquivo `src/main/java/br/edu/upf/minha_api/users/UserController.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.Valid;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/users")
@Tag(name = "users", description = "Operacoes de usuarios")
public class UserController {
  private final UserService service;

  public UserController(UserService service) {
    this.service = service;
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  @Operation(summary = "Criar usuario")
  public User create(@Valid @RequestBody CreateUserRequest request) {
    return service.create(request);
  }

  @GetMapping
  @Operation(summary = "Listar usuarios")
  public List<User> findAll() {
    return service.findAll();
  }

  @GetMapping("/{id}")
  @Operation(summary = "Buscar usuario por ID")
  public User findOne(@PathVariable String id) {
    return service.findOne(id);
  }

  @PutMapping("/{id}")
  @Operation(summary = "Atualizar usuario")
  public User update(@PathVariable String id, @Valid @RequestBody UpdateUserRequest request) {
    return service.update(id, request);
  }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  @Operation(summary = "Deletar usuario")
  public void remove(@PathVariable String id) {
    service.remove(id);
  }
}
```

`@RestController` transforma a classe em um controller REST.

`@RequestMapping("/users")` define a rota base usada pelo projeto.

`@Valid` ativa as validações colocadas nos DTOs.

O controller não acessa o repository. Ele chama o service. Isso mantém o controller simples.

### 12. Conferir a classe principal

O Spring Initializr já criou a classe principal.

Ela deve estar em `src/main/java/br/edu/upf/minha_api/MinhaApiApplication.java`:

```java
package br.edu.upf.minha_api;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MinhaApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(MinhaApiApplication.class, args);
    }
}
```

`@SpringBootApplication` liga a configuração automática do Spring Boot e faz a aplicação procurar componentes como controllers, services e repositories dentro do pacote `br.edu.upf.minha_api`.

### 13. Rodar a API localmente

Com o MongoDB rodando, execute:

```bash
./gradlew bootRun
```

No Windows, se estiver usando PowerShell ou CMD:

```bash
gradlew.bat bootRun
```

A API ficará disponível em:

```text
http://localhost:8080
```

### 14. Testar pelo Swagger

Abra:

```text
http://localhost:8080/swagger-ui.html
```

O Swagger mostra os endpoints disponíveis e permite testar a API pelo navegador.

Para criar um usuário, use o endpoint `POST /users` com este corpo:

```json
{
  "name": "Bruno",
  "email": "bruno@example.com"
}
```

Resposta esperada:

```json
{
  "id": "665f1b1f9c7a4d31d4d0a111",
  "name": "Bruno",
  "email": "bruno@example.com",
  "createdAt": "2026-05-03T21:00:00Z"
}
```

O `id` e o `createdAt` serão diferentes na sua máquina.

Para listar:

```text
GET /users
```

Para buscar por id:

```text
GET /users/{id}
```

Para atualizar:

```text
PUT /users/{id}
```

Com corpo:

```json
{
  "name": "Bruno Silva",
  "email": "bruno.silva@example.com"
}
```

Para remover:

```text
DELETE /users/{id}
```

Se remover com sucesso, a resposta será `204 No Content`.

### 15. Testar validações e erros comuns

Tente criar um usuário sem nome:

```json
{
  "name": "",
  "email": "bruno@example.com"
}
```

A API deve retornar erro `400 Bad Request`, porque `name` tem `@NotBlank`.

Tente criar um usuário com email inválido:

```json
{
  "name": "Bruno",
  "email": "email-invalido"
}
```

A API também deve retornar `400 Bad Request`, porque `email` tem `@Email`.

Tente criar dois usuários com o mesmo email. A segunda tentativa deve retornar `409 Conflict`.

### 16. Criar o Dockerfile

Agora vamos preparar a API para rodar em container.

Crie o arquivo `Dockerfile` na raiz do projeto:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build

WORKDIR /app

COPY gradlew .
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

RUN chmod +x gradlew

COPY src ./src
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Esse Dockerfile tem duas etapas.

A primeira usa JDK 17 para compilar a aplicação.

A segunda usa JRE 17 para executar o `.jar`. Isso deixa a imagem final menor.

### 17. Criar o .dockerignore

Crie o arquivo `.dockerignore`:

```dockerignore
build/
.gradle/
.git/
.idea/
.vscode/
*.log
```

Esse arquivo evita copiar para a imagem arquivos que não são necessários para executar a aplicação.

### 18. Criar o Docker Compose

Crie o arquivo `docker-compose.yml` na raiz do projeto:

```yaml
services:
  mongo:
    image: mongo:7
    container_name: mongo-api
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  api:
    build: .
    container_name: minha-api
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      MONGODB_URI: mongodb://mongo:27017/minha_api
    depends_on:
      - mongo

volumes:
  mongo_data:
```

Quando a API roda fora do Docker, ela usa `localhost`.

Quando a API roda dentro do Docker Compose, `localhost` seria o próprio container da API. Por isso usamos `mongodb://mongo:27017/minha_api`. O nome `mongo` vem do serviço definido no compose.

Execute:

```bash
docker compose up --build
```

A API continuará disponível em:

```text
http://localhost:8080/users
```

O Swagger continuará disponível em:

```text
http://localhost:8080/swagger-ui.html
```

Para parar tudo:

```bash
docker compose down
```

Se quiser apagar também o volume com os dados:

```bash
docker compose down -v
```

## Código completo

Confira os arquivos principais da aplicação.

`build.gradle`:

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

`src/main/resources/application.properties`:

```properties
spring.application.name=minha-api
server.port=${PORT:8080}

spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/minha_api}

springdoc.swagger-ui.path=/swagger-ui.html
```

`src/main/java/br/edu/upf/minha_api/users/User.java`:

```java
package br.edu.upf.minha_api.users;

import java.time.Instant;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@NoArgsConstructor
@Document(collection = "users")
public class User {
  @Id
  private String id;

  private String name;

  @Indexed(unique = true)
  private String email;

  private Instant createdAt = Instant.now();
}
```

`src/main/java/br/edu/upf/minha_api/users/UserRepository.java`:

```java
package br.edu.upf.minha_api.users;

import java.util.Optional;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface UserRepository extends MongoRepository<User, String> {
  Optional<User> findByEmail(String email);
}
```

`src/main/java/br/edu/upf/minha_api/users/CreateUserRequest.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record CreateUserRequest(
  @NotBlank(message = "Nome e obrigatorio")
  String name,

  @NotBlank(message = "Email e obrigatorio")
  @Email(message = "Email invalido")
  String email
) {}
```

`src/main/java/br/edu/upf/minha_api/users/UpdateUserRequest.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.constraints.Email;

public record UpdateUserRequest(
  String name,

  @Email(message = "Email invalido")
  String email
) {}
```

`src/main/java/br/edu/upf/minha_api/users/UserService.java`:

```java
package br.edu.upf.minha_api.users;

import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class UserService {
  private final UserRepository repository;

  public UserService(UserRepository repository) {
    this.repository = repository;
  }

  public User create(CreateUserRequest request) {
    var existingUser = repository.findByEmail(request.email());

    if (existingUser.isPresent()) {
      throw new ResponseStatusException(HttpStatus.CONFLICT, "Email ja cadastrado");
    }

    var user = new User();
    user.setName(request.name());
    user.setEmail(request.email());

    return repository.save(user);
  }

  public List<User> findAll() {
    return repository.findAll();
  }

  public User findOne(String id) {
    return repository.findById(id)
      .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Usuario nao encontrado"));
  }

  public User update(String id, UpdateUserRequest request) {
    var user = findOne(id);

    updateName(user, request.name());
    updateEmail(user, request.email());

    return repository.save(user);
  }

  public void remove(String id) {
    var user = findOne(id);
    repository.delete(user);
  }

  private void updateName(User user, String name) {
    if (name == null || name.isBlank()) {
      return;
    }

    user.setName(name);
  }

  private void updateEmail(User user, String email) {
    if (email == null || email.isBlank()) {
      return;
    }

    var existingUser = repository.findByEmail(email);

    if (existingUser.isEmpty()) {
      user.setEmail(email);
      return;
    }

    if (existingUser.get().getId().equals(user.getId())) {
      user.setEmail(email);
      return;
    }

    throw new ResponseStatusException(HttpStatus.CONFLICT, "Email ja cadastrado");
  }
}
```

`src/main/java/br/edu/upf/minha_api/users/UserController.java`:

```java
package br.edu.upf.minha_api.users;

import jakarta.validation.Valid;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import java.util.List;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/users")
@Tag(name = "users", description = "Operacoes de usuarios")
public class UserController {
  private final UserService service;

  public UserController(UserService service) {
    this.service = service;
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  @Operation(summary = "Criar usuario")
  public User create(@Valid @RequestBody CreateUserRequest request) {
    return service.create(request);
  }

  @GetMapping
  @Operation(summary = "Listar usuarios")
  public List<User> findAll() {
    return service.findAll();
  }

  @GetMapping("/{id}")
  @Operation(summary = "Buscar usuario por ID")
  public User findOne(@PathVariable String id) {
    return service.findOne(id);
  }

  @PutMapping("/{id}")
  @Operation(summary = "Atualizar usuario")
  public User update(@PathVariable String id, @Valid @RequestBody UpdateUserRequest request) {
    return service.update(id, request);
  }

  @DeleteMapping("/{id}")
  @ResponseStatus(HttpStatus.NO_CONTENT)
  @Operation(summary = "Deletar usuario")
  public void remove(@PathVariable String id) {
    service.remove(id);
  }
}
```

## Erros comuns

### MongoDB não está rodando

Se a aplicação não conseguir conectar no banco, confira:

```bash
docker ps
```

Se o container estiver parado:

```bash
docker start mongo-api
```

### Porta 27017 já está em uso

Isso significa que já existe outro MongoDB usando a porta. Você pode parar o container antigo ou mudar a porta no comando Docker.

### Swagger não abre

Confira se a dependência `springdoc-openapi-starter-webmvc-ui` está no `build.gradle`.

Depois reinicie a aplicação:

```bash
./gradlew bootRun
```

### Erro ao usar ./gradlew no Linux

Se aparecer erro de permissão:

```bash
chmod +x gradlew
./gradlew bootRun
```

### Email duplicado não dá conflito

O índice único pode não existir ainda se a coleção já foi criada antes. Para uma aula local, a solução mais simples é apagar o volume e subir de novo:

```bash
docker compose down -v
docker compose up --build
```

Em produção, essa decisão precisa ser feita com cuidado para não apagar dados reais.

## Resumo

Nesta aula você criou uma API REST com Java 17, Spring Boot e MongoDB.

Você viu como organizar a aplicação por recurso, mantendo as responsabilidades claras:

* controller para receber HTTP;
* service para regra de negócio;
* model para representar o documento do MongoDB;
* repository para acessar o banco;
* DTO para entrada da API.

Também configurou Swagger, validação com Bean Validation, MongoDB via Docker e execução com Docker Compose.

## Próximo passo

Na próxima aula faz sentido evoluir esse projeto com princípios de organização de código e SOLID.

Agora que a API já funciona, fica mais fácil discutir por que separar responsabilidades, como evitar services grandes demais e quando uma interface realmente ajuda.

## Referências

* [Spring Initializr](https://start.spring.io/)
* [Spring Boot](https://spring.io/projects/spring-boot)
* [Spring Data MongoDB](https://spring.io/projects/spring-data-mongodb)
* [Springdoc OpenAPI](https://springdoc.org/)
* [MongoDB](https://www.mongodb.com/docs/)

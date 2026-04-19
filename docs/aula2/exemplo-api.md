# Aula 2 — Exemplo Prático de API com Spring Boot e MongoDB

## Objetivo da Aula

Nesta aula vamos **colocar em prática** os conceitos vistos sobre APIs.
A ideia é construir uma **API simples**, usando **Java**, **Spring Boot** e **MongoDB**.
Essa API permitirá **criar, listar, buscar, atualizar e deletar usuários** — um CRUD básico.

## Stack Utilizada

- **Java 21** → linguagem usada para construir a API.
- **Spring Boot** → framework para criar aplicações Java com configuração simplificada.
- **Spring Web** → módulo para criar endpoints HTTP/REST.
- **Spring Data MongoDB** → integração entre Spring Boot e MongoDB.
- **Validation** → validação dos dados enviados nas requisições.
- **Lombok** → reduz código repetitivo como getters, setters e construtores.
- **Swagger/OpenAPI** → documentação visual e testável da API.
- **springdoc-openapi** → biblioteca que integra Swagger UI com Spring Boot.
- **MongoDB** → banco NoSQL orientado a documentos.
- **Docker** → usado para subir o MongoDB rapidamente.
- **Gradle Wrapper** → ferramenta de build gerada junto com o projeto.

## Antes de Começar

Você precisa ter instalado:

- **Java JDK 21**
- **Docker Desktop** ou **Docker Engine**
- **Editor de código**: IntelliJ IDEA, VS Code ou Eclipse
- **Navegador web**
- **Postman** ou **Insomnia** para testar requisições `POST`, `PUT` e `DELETE`

Links úteis:

- Java JDK 21: [https://adoptium.net/temurin/releases/?version=21](https://adoptium.net/temurin/releases/?version=21)
- Docker Desktop: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
- IntelliJ IDEA Community: [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/)
- Visual Studio Code: [https://code.visualstudio.com/](https://code.visualstudio.com/)
- Postman: [https://www.postman.com/downloads/](https://www.postman.com/downloads/)
- Insomnia: [https://insomnia.rest/download](https://insomnia.rest/download)

> O Gradle será gerado junto com o projeto pelo Spring Initializr, usando o Gradle Wrapper. Não é obrigatório instalar Gradle manualmente para esta aula.

## Padrão de Código da Aula

Para manter o código mais limpo:

- Use `var` para variáveis locais quando o tipo estiver claro pelo lado direito da atribuição.
- Use **early return** para evitar `if` aninhado desnecessário.
- Separe regras auxiliares em métodos privados quando isso deixar o service mais legível.
- Deixe o controller fino: ele recebe HTTP e delega a regra para o service.

## Estrutura da Aplicação

```
minha-api/
├─ src/
│  └─ main/
│     ├─ java/
│     │  └─ br/edu/upf/minhaapi/
│     │     ├─ MinhaApiApplication.java
│     │     ├─ config/
│     │     │  └─ OpenApiConfig.java
│     │     └─ users/
│     │        ├─ User.java
│     │        ├─ UserRepository.java
│     │        ├─ UserService.java
│     │        ├─ UserController.java
│     │        ├─ CreateUserRequest.java
│     │        └─ UpdateUserRequest.java
│     └─ resources/
│        └─ application.properties
├─ build.gradle
├─ settings.gradle
├─ gradlew
├─ gradle/
├─ Dockerfile
└─ docker-compose.yml
```

## Passo 1 — Subir o MongoDB com Docker

Execute o MongoDB localmente usando Docker:

```bash
docker run --name mongo-api \
  -p 27017:27017 \
  -d mongo:7
```

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

# Ver logs
docker logs mongo-api
```

## Passo 2 — Criar o Projeto no Spring Initializr

Abra o site [https://start.spring.io/](https://start.spring.io/) no navegador.

Configure o projeto com estas opções:

| Campo | Valor |
|-------|-------|
| **Project** | Gradle - Groovy |
| **Language** | Java |
| **Spring Boot** | 3.5.x |
| **Group** | `br.edu.upf` |
| **Artifact** | `minha-api` |
| **Name** | `minha-api` |
| **Description** | `API Spring Boot com MongoDB` |
| **Package name** | `br.edu.upf.minhaapi` |
| **Packaging** | Jar |
| **Java** | 21 |

Em **Dependencies**, clique em **Add Dependencies** e adicione:

- **Spring Web**
- **Spring Data MongoDB**
- **Validation**
- **Lombok**

Depois clique em **Generate** para baixar o arquivo `.zip` do projeto.

> O Swagger será adicionado manualmente no `build.gradle`, porque ele usa a biblioteca `springdoc-openapi`.

## Passo 3 — Descompactar e Abrir o Projeto

```bash
# Descompactar o projeto
unzip minha-api.zip -d minha-api

# Entrar no diretório
cd minha-api
```

Abra a pasta `minha-api` no seu editor de código.

## Passo 4 — Configurar a Conexão com o MongoDB

Edite o arquivo `src/main/resources/application.properties`:

```properties
spring.application.name=minha-api
server.port=${PORT:8080}
spring.data.mongodb.uri=${MONGODB_URI:mongodb://localhost:27017/minha_api}
springdoc.swagger-ui.path=/swagger-ui.html
```

Essa configuração usa:

- `localhost:27017` quando a API roda direto na sua máquina.
- `MONGODB_URI` quando a API roda em container ou ambiente de produção.
- `minha_api` como nome do banco.
- `/swagger-ui.html` como endereço da documentação visual da API.

## Passo 5 — Adicionar Swagger no Gradle

Abra o arquivo `build.gradle` e adicione a dependência do `springdoc-openapi` dentro do bloco `dependencies`.

O trecho de configurações e dependências ficará parecido com este:

```groovy
configurations {
  compileOnly {
    extendsFrom annotationProcessor
  }
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.17'

  compileOnly 'org.projectlombok:lombok'
  annotationProcessor 'org.projectlombok:lombok'

  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

Essa dependência libera dois endereços importantes:

- `http://localhost:8080/swagger-ui.html` → tela visual para testar a API.
- `http://localhost:8080/v3/api-docs` → contrato OpenAPI em JSON.

> Se o editor mostrar erro nos métodos gerados pelo Lombok, habilite **annotation processing** nas configurações da IDE.

## Passo 6 — Configurar Informações do Swagger

Crie a pasta `src/main/java/br/edu/upf/minhaapi/config`.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/config/OpenApiConfig.java`:

```java
package br.edu.upf.minhaapi.config;

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

## Passo 7 — Criar o Modelo de Usuário

Crie a pasta `src/main/java/br/edu/upf/minhaapi/users`.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/User.java`:

```java
package br.edu.upf.minhaapi.users;

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

## Passo 8 — Criar os DTOs

DTOs organizam os dados que entram na API.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/CreateUserRequest.java`:

```java
package br.edu.upf.minhaapi.users;

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

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/UpdateUserRequest.java`:

```java
package br.edu.upf.minhaapi.users;

import jakarta.validation.constraints.Email;

public record UpdateUserRequest(
  String name,

  @Email(message = "Email invalido")
  String email
) {}
```

## Passo 9 — Criar o Repository

O repository é a camada que conversa com o MongoDB.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/UserRepository.java`:

```java
package br.edu.upf.minhaapi.users;

import java.util.Optional;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface UserRepository extends MongoRepository<User, String> {
  Optional<User> findByEmail(String email);
}
```

## Passo 10 — Criar o Service

O service concentra as regras da aplicação.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/UserService.java`:

```java
package br.edu.upf.minhaapi.users;

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

## Passo 11 — Criar o Controller

O controller expõe os endpoints HTTP da API.

Crie o arquivo `src/main/java/br/edu/upf/minhaapi/users/UserController.java`:

```java
package br.edu.upf.minhaapi.users;

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

## Passo 12 — Rodar a API Localmente

Com o MongoDB rodando no Docker, execute a API:

```bash
./gradlew bootRun
```

A API ficará disponível em:

```text
http://localhost:8080
```

## Passo 13 — Testar a API

Use o Swagger, Postman ou Insomnia.

### Acessar Swagger

Abra no navegador:

```text
http://localhost:8080/swagger-ui.html
```

No Swagger você pode:

- Ver todos os endpoints disponíveis.
- Testar `POST`, `GET`, `PUT` e `DELETE`.
- Conferir os modelos de entrada e saída.
- Ver os status HTTP esperados.

### Criar Usuário

**Método:** `POST`

**URL:** `http://localhost:8080/users`

**Body JSON:**

```json
{
  "name": "Bruno",
  "email": "bruno@example.com"
}
```

### Listar Usuários

**Método:** `GET`

**URL:** `http://localhost:8080/users`

Também é possível abrir essa URL no navegador.

### Buscar Usuário por ID

**Método:** `GET`

**URL:** `http://localhost:8080/users/ID_DO_USUARIO`

Troque `ID_DO_USUARIO` pelo `id` retornado ao criar ou listar usuários.

### Atualizar Usuário

**Método:** `PUT`

**URL:** `http://localhost:8080/users/ID_DO_USUARIO`

**Body JSON:**

```json
{
  "name": "Bruno Silva",
  "email": "bruno.silva@example.com"
}
```

### Deletar Usuário

**Método:** `DELETE`

**URL:** `http://localhost:8080/users/ID_DO_USUARIO`

Se a operação funcionar, a API retorna status `204 No Content`.

## Passo 14 — Dockerizar a API

Crie o arquivo `Dockerfile` na raiz do projeto:

```dockerfile
# Etapa 1: construir a aplicacao com Gradle Wrapper e JDK 21
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

COPY gradlew .
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

RUN chmod +x gradlew

COPY src ./src
RUN ./gradlew bootJar --no-daemon

# Etapa 2: executar apenas com JRE
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Crie o arquivo `.dockerignore`:

```dockerignore
build/
.gradle/
.git/
.idea/
.vscode/
*.log
```

## Passo 15 — Criar o Docker Compose

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

## Dicas Importantes

- Use o `Controller` para receber requisições HTTP.
- Use o `Service` para concentrar regras de negócio.
- Use o `Repository` para acessar o MongoDB.
- Use DTOs para controlar os dados aceitos pela API.
- Valide os dados de entrada antes de salvar no banco.
- Em MongoDB, o `id` normalmente é uma `String`, não um número sequencial.

## Referências

- [Spring Initializr](https://start.spring.io/)
- [Documentação Spring Boot](https://spring.io/projects/spring-boot)
- [Documentação Spring Data MongoDB](https://spring.io/projects/spring-data-mongodb)
- [Documentação MongoDB](https://www.mongodb.com/docs/)

---

**Fim da Aula 2** — Agora você tem uma **API funcional com Spring Boot e MongoDB** para evoluir nas próximas aulas.

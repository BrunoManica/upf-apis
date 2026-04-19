# Docker

## Visão Geral

O **Docker** é uma plataforma de containerização que revolucionou a forma como desenvolvemos, empacotamos e implantamos aplicações. Ele resolve um dos maiores problemas da área de desenvolvimento: **"funciona na minha máquina"**.

### Por que Docker?
- **Consistência**: Aplicação funciona igual em qualquer ambiente
- **Isolamento**: Cada aplicação roda em seu próprio ambiente
- **Portabilidade**: Move facilmente entre desenvolvimento, teste e produção
- **Eficiência**: Compartilha recursos do sistema operacional
- **Escalabilidade**: Fácil de replicar e escalar horizontalmente

## Conceitos Fundamentais

### Dockerfile
**Definição:** Arquivo de texto com instruções para construir uma imagem Docker.

**Função:** Define exatamente como criar o ambiente da aplicação:
- Sistema operacional base
- Dependências e bibliotecas
- Configurações do sistema
- Comandos de inicialização

**Analogia:** É como uma **receita de bolo** - você lista todos os ingredientes e o passo a passo para fazer o bolo.

### Imagem
**Definição:** Template imutável e somente leitura usado para criar containers.

**Características:**
- **Imutável**: Não pode ser alterada após criada
- **Versionada**: Cada versão tem uma tag única
- **Layered**: Composta por camadas que otimizam armazenamento
- **Portátil**: Funciona em qualquer sistema com Docker

**Analogia:** É como um **molde de bolo** - você pode usar o mesmo molde para fazer vários bolos idênticos.

### Container
**Definição:** Instância em execução de uma imagem Docker.

**Características:**
- **Isolado**: Tem seu próprio sistema de arquivos e processos
- **Efêmero**: Pode ser criado e destruído rapidamente
- **Leve**: Compartilha o kernel do sistema operacional host
- **Portátil**: Roda em qualquer lugar que tenha Docker

**Analogia:** É como um **bolo sendo servido** - é a instância real do molde (imagem) em uso.

| Conceito       | Analogia                         | Explicação                                                                                                                                                                                          |
| -------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Dockerfile** | **A receita do bolo**         | É o arquivo com o passo a passo de como preparar o bolo — quais ingredientes usar, em que ordem, e como misturar e assar.                                                                           |
| **Imagem**     | **O molde do bolo**    | É o resultado de seguir a receita (Dockerfile). É o *template pronto* que pode ser replicado. Você ainda não está comendo — apenas tem o molde "feito" e guardado. |
| **Container**  | **Um bolo sendo servido** | É a instância da imagem em execução. Cada container é um bolo "sendo servido e comido" — ou seja, a aplicação rodando. Você pode fazer vários bolos (containers) do mesmo molde (imagem).          |

## Como o Docker Funciona?

### Arquitetura do Docker

```
┌─────────────────────────────────────┐
│        Docker Client (CLI)          │ ← Você digita comandos aqui
├─────────────────────────────────────┤
│        Docker Daemon                │ ← Gerencia tudo
├─────────────────────────────────────┤
│        Docker Registry              │ ← Armazena imagens
├─────────────────────────────────────┤
│        Containers                   │ ← Suas aplicações rodando
│  ┌─────────┬─────────┬─────────┐    │
│  │Container│Container│Container│    │
│  │  App A  │  App B  │  App C  │    │
│  └─────────┴─────────┴─────────┘    │
├─────────────────────────────────────┤
│        Sistema Operacional          │ ← Linux/Windows/macOS
└─────────────────────────────────────┘
```

### Fluxo de Funcionamento

1. **Você escreve** um Dockerfile
2. **Docker constrói** uma imagem baseada no Dockerfile
3. **Docker executa** um container a partir da imagem
4. **Sua aplicação roda** isolada no container

## Componentes Principais

### **Docker Engine**
O coração do Docker, composto por:

**Docker Daemon (dockerd):** 
  - Processo que roda em background
  - Gerencia imagens, containers, volumes e redes
  - Escuta comandos via API REST

**Docker CLI (docker):** 
  - Interface de linha de comando
  - Envia comandos para o daemon
  - O que você usa no terminal

**Docker API:** 
  - Interface REST para automação
  - Usada por ferramentas como Docker Compose

### **Docker Registry**
Sistema de armazenamento e distribuição de imagens:

**Docker Hub:** 
  - Registro público oficial
  - Milhares de imagens prontas
  - Ex: `nginx`, `postgres`, `eclipse-temurin`

**Registros Privados:** 
  - Para empresas (AWS ECR, Azure ACR)
  - Imagens internas e confidenciais
  - Controle de acesso

### **Docker Compose**
Ferramenta para orquestrar múltiplos containers:

- **Arquivo:** `docker-compose.yml`
- **Função:** Define serviços, redes e volumes
- **Vantagem:** Um comando para subir toda a aplicação

## Comandos Essenciais

### Teste Inicial
Verifique se o Docker está funcionando:

```bash
docker run hello-world
```

###  Gerenciamento de Imagens

```bash
# Baixar uma imagem do Docker Hub
docker pull postgres:15-alpine

# Listar imagens locais
docker images

# Construir uma imagem a partir do Dockerfile  
docker build -t minha-api:v1.0 .

# Remover uma imagem
docker rmi postgres:15-alpine

# Ver histórico de uma imagem
docker history postgres:15-alpine
```

### Gerenciamento de Containers

```bash
# Executar um container (modo interativo)
docker run -it ubuntu:latest bash

# Executar PostgreSQL em background
docker run -d -p 5432:5432 \
  --name meu-postgres \
  -e POSTGRES_PASSWORD=senha123 \
  -e POSTGRES_DB=meubanco \
  postgres:15-alpine

# Listar containers rodando
docker ps

# Listar todos os containers (incluindo parados)
docker ps -a

# Parar um container
docker stop meu-postgres

# Iniciar um container parado
docker start meu-postgres

# Remover um container
docker rm meu-postgres

# Executar comando em container rodando
docker exec -it meu-postgres psql -U postgres

# Ver logs do container
docker logs meu-postgres
```

### Docker Compose

```bash
# Iniciar todos os serviços
docker-compose up

# Iniciar em background
docker-compose up -d

# Parar todos os serviços
docker-compose down

# Ver logs dos serviços
docker-compose logs

# Reconstruir e iniciar
docker-compose up --build
```

### Comandos de Manutenção

```bash
# Ver uso de recursos
docker stats

# Limpar containers parados
docker container prune

# Limpar imagens não utilizadas
docker image prune

# Limpar tudo (cuidado!)
docker system prune -a

# Copiar arquivos entre host e container
docker cp arquivo.txt container:/caminho/
docker cp container:/caminho/arquivo.txt ./
```

## Exemplos Práticos

### Exemplo 1: API Spring Boot Simples

Vamos criar uma API básica com Java e Spring Boot:

**Antes de começar, você precisa ter instalado:**

- **Java JDK 21**: necessário para desenvolver e compilar a aplicação.
- **Docker Desktop** ou **Docker Engine**: necessário para construir e executar a imagem.
- **Editor de código**: IntelliJ IDEA, VS Code ou Eclipse.
- **Navegador web**: usado para acessar o Spring Initializr.
- **Ferramenta para descompactar `.zip`**: o próprio sistema operacional normalmente já faz isso.

> O Gradle será gerado junto com o projeto pelo Spring Initializr, usando o Gradle Wrapper. Por isso, não é obrigatório instalar Gradle manualmente para esta aula.

**Links para baixar:**

- Java JDK 21: [https://adoptium.net/temurin/releases/?version=21](https://adoptium.net/temurin/releases/?version=21)
- Docker Desktop: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
- IntelliJ IDEA Community: [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/)
- Visual Studio Code: [https://code.visualstudio.com/](https://code.visualstudio.com/)

**1. Criar o projeto no Spring Initializr:**

Abra o site [https://start.spring.io/](https://start.spring.io/) no navegador.

Configure o projeto com estas opções:

| Campo | Valor |
|-------|-------|
| **Project** | Gradle - Groovy |
| **Language** | Java |
| **Spring Boot** | versão estável sugerida pelo site |
| **Group** | `br.edu.upf` |
| **Artifact** | `api-spring-docker` |
| **Name** | `api-spring-docker` |
| **Description** | `API Spring Boot com Docker` |
| **Package name** | `br.edu.upf.apispringdocker` |
| **Packaging** | Jar |
| **Java** | 21 |

Em **Dependencies**, clique em **Add Dependencies** e adicione:

- **Spring Web**: permite criar endpoints HTTP/REST com Spring MVC e servidor Tomcat embutido.

Depois clique em **Generate** para baixar o arquivo `.zip` do projeto.

**2. Descompactar e abrir o projeto:**

```bash
# Descompactar o projeto
unzip api-spring-docker.zip -d api-spring-docker

# Entrar no diretório
cd api-spring-docker
```

Abra a pasta `api-spring-docker` no seu editor de código.

**3. Arquivos customizados (apenas o que você modifica):**

**src/main/java/br/edu/upf/apispringdocker/ApiSpringDockerApplication.java:**
```java
package br.edu.upf.apispringdocker;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiSpringDockerApplication {
  public static void main(String[] args) {
    SpringApplication.run(ApiSpringDockerApplication.class, args);
  }
}
```

**src/main/java/br/edu/upf/apispringdocker/DockerController.java:**
```java
package br.edu.upf.apispringdocker;

import java.time.Instant;
import java.util.Map;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DockerController {
  @GetMapping("/")
  public String hello() {
    return "Ola Docker! API Spring Boot rodando em container.";
  }

  @GetMapping("/health")
  public Map<String, String> health() {
    return Map.of(
      "status", "OK",
      "message", "API funcionando perfeitamente!",
      "timestamp", Instant.now().toString(),
      "environment", "Docker Container"
    );
  }
}
```

**src/main/resources/application.properties:**
```properties
server.port=${PORT:8080}
```

**Dockerfile:**
```dockerfile
# Etapa 1: construir a aplicacao com Gradle Wrapper e JDK 21
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Copiar arquivos do Gradle primeiro para aproveitar cache
COPY gradlew .
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

# Garantir permissao de execucao no wrapper
RUN chmod +x gradlew

# Copiar codigo-fonte e gerar o .jar
COPY src ./src
RUN ./gradlew bootJar --no-daemon

# Etapa 2: executar apenas com JRE
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

# Expor porta
EXPOSE 8080

# Comando para iniciar em produção
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**.dockerignore:**
```dockerignore
build/
.gradle/
.git/
.idea/
.vscode/
*.log
```

**Build e Execução:**
```bash
# Construir a imagem
docker build -t api-spring:latest .

# Executar o container
docker run -d -p 8080:8080 --name api-spring api-spring:latest

# Testar a API no navegador
# Abra:
# http://localhost:8080
# http://localhost:8080/health
```

**Estrutura de arquivos (após criar com Spring Initializr):**
```
api-spring-docker/
├── src/
│   └── main/
│       ├── java/
│       │   └── br/edu/upf/apispringdocker/
│       │       ├── ApiSpringDockerApplication.java ← Gerado automaticamente
│       │       └── DockerController.java           ← Criado por você
│       └── resources/
│           └── application.properties              ← Modificado
├── build.gradle                                 ← Gerado automaticamente
├── settings.gradle                              ← Gerado automaticamente
├── gradlew                                      ← Gerado automaticamente
├── gradle/                                      ← Gerado automaticamente
├── Dockerfile                                   ← Criado por você
└── .dockerignore                                ← Criado por você
```


## Vantagens do Docker

### Para Desenvolvedores
- **Consistência**: Elimina o "funciona na minha máquina"
- **Setup rápido**: Ambiente pronto em minutos
- **Isolamento**: Cada projeto tem suas dependências
- **Versionamento**: Controle de versões do ambiente
- **Portabilidade**: Move entre máquinas facilmente

### Para Operações (DevOps)
- **Deploy consistente**: Mesmo ambiente em dev/test/prod
- **Escalabilidade**: Fácil de replicar e escalar
- **Eficiência**: Melhor uso de recursos do servidor
- **Rollback**: Volta versão anterior rapidamente
- **Automação**: Integra com CI/CD facilmente

### Para Organizações
- **Padronização**: Todos usam o mesmo ambiente
- **Produtividade**: Menos tempo configurando, mais desenvolvendo
- **Custos**: Reduz necessidade de hardware dedicado
- **Qualidade**: Menos bugs relacionados ao ambiente

## Casos de Uso Práticos

### Desenvolvimento
- **Ambientes isolados**: Cada desenvolvedor tem seu ambiente
- **Testes automatizados**: Containers para testes CI/CD
- **Dependências complexas**: Redis, PostgreSQL, Elasticsearch

### Produção
- **Microserviços**: Cada serviço em seu container
- **Escalabilidade**: Kubernetes orquestra containers
- **Migração**: Aplicações legadas em containers

### DevOps
- **Infraestrutura como código**: Dockerfiles versionados
- **Ambientes múltiplos**: dev, staging, produção
- **Backup**: Imagens como backup do ambiente

## Alternativas ao Docker

### Podman
- **Características**: Daemonless, rootless, compatível com Docker
- **Vantagens**: Mais seguro, não precisa de daemon
- **Uso**: Comandos similares ao Docker

### Containerd
- **Características**: Runtime de containers usado pelo Kubernetes
- **Vantagens**: Mais leve, focado apenas em runtime
- **Uso**: Gerenciado pelo Kubernetes

### LXC/LXD
- **Características**: Containers de sistema operacional
- **Vantagens**: Mais isolamento, sistema completo
- **Uso**: Virtualização a nível de sistema

## Ferramentas Úteis

### Docker Desktop
Interface gráfica oficial para gerenciar containers, imagens e volumes.

### Lazydocker
Interface de terminal (TUI) para gerenciar Docker de forma visual no terminal.

### Portainer
Interface web para gerenciar ambientes Docker em produção.

## Quando Usar Docker?

### ✅ Use Docker quando:
- Desenvolvimento de APIs modernas (Spring Boot, Quarkus, Micronaut)
- Microserviços
- CI/CD pipelines
- Aplicações stateless
- Escalabilidade horizontal é importante
- Portabilidade entre ambientes é crítica
- Desenvolvimento com Java e outras stacks de back-end

### ❌ Evite Docker quando:
- Aplicações com GUI complexa
- Performance crítica (jogos, HPC)
- Aplicações que precisam de kernel customizado
- Sistemas legados muito complexos

## Referências

Este conteúdo é baseado na documentação oficial do [Docker](https://docs.docker.com/).

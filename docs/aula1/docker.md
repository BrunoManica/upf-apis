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

- **Docker Daemon (dockerd):** 
  - Processo que roda em background
  - Gerencia imagens, containers, volumes e redes
  - Escuta comandos via API REST

- **Docker CLI (docker):** 
  - Interface de linha de comando
  - Envia comandos para o daemon
  - O que você usa no terminal

- **Docker API:** 
  - Interface REST para automação
  - Usada por ferramentas como Docker Compose

### **Docker Registry**
Sistema de armazenamento e distribuição de imagens:

- **Docker Hub:** 
  - Registro público oficial
  - Milhares de imagens prontas
  - Ex: `nginx`, `postgres`, `node`

- **Registros Privados:** 
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

### Gerenciamento de Imagens

```bash
# Baixar uma imagem do Docker Hub
docker pull postgres:15-alpine

# Listar imagens locais
docker images

# Construir uma imagem a partir do Dockerfile
docker build -t minha-app:v1.0 .

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

### Exemplo 1: API NestJS Simples

Vamos criar uma API básica com NestJS:

**1. Criar o projeto NestJS:**
```bash
# Instalar NestJS CLI globalmente
npm install -g @nestjs/cli

# Criar novo projeto
nest new api-nestjs-docker

# Entrar no diretório
cd api-nestjs-docker
```

**2. Arquivos customizados (apenas o que você modifica):**

**src/app.controller.ts:**
```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get('health')
  getHealth() {
    return {
      status: 'OK',
      message: 'API funcionando perfeitamente!',
      timestamp: new Date().toISOString(),
      environment: 'Docker Container'
    };
  }
}
```

**src/app.service.ts:**
```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Olá Docker! API NestJS rodando em container 🐳';
  }
}
```

**src/main.ts:**
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Habilitar CORS
  app.enableCors();
  
  const port = process.env.PORT || 3000;
  await app.listen(port);
  
  console.log(`🚀 API rodando na porta ${port}`);
}
bootstrap();
```

**Dockerfile:**
```dockerfile
# Usar imagem oficial do Node.js
FROM node:18-alpine

# Instalar NestJS CLI globalmente
RUN npm install -g @nestjs/cli

# Definir diretório de trabalho
WORKDIR /app

# Copiar arquivos de dependências
COPY package*.json ./

# Instalar dependências
RUN npm install

# Copiar código da aplicação
COPY . .

# Compilar a aplicação
RUN npm run build

# Expor porta
EXPOSE 3000

# Comando para iniciar em produção
CMD ["npm", "run", "start:prod"]
```

**Build e Execução:**
```bash
# Construir a imagem
docker build -t api-nestjs:latest .

# Executar o container
docker run -d -p 3000:3000 --name api-nestjs api-nestjs

# Testar a API
curl http://localhost:3000
curl http://localhost:3000/health
```

**Estrutura de arquivos (após criar com NestJS CLI):**
```
api-nestjs-docker/
├── src/
│   ├── main.ts          ← Modificado
│   ├── app.module.ts    ← Gerado automaticamente
│   ├── app.controller.ts ← Modificado
│   └── app.service.ts   ← Modificado
├── package.json         ← Gerado automaticamente
├── Dockerfile           ← Criado por você
└── .dockerignore        ← Criado por você
```

### Exemplo 2: API NestJS com Banco de Dados

Vamos criar uma API completa com NestJS e PostgreSQL usando Docker Compose:

**1. Criar projeto e instalar dependências:**
```bash
# Criar projeto NestJS
nest new api-nestjs-postgres
cd api-nestjs-postgres

# Instalar dependências do banco
npm install @nestjs/typeorm @nestjs/config typeorm pg
npm install -D @types/pg
```

**2. Arquivos customizados:**

**src/app.module.ts:**
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule } from '@nestjs/config';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    ConfigModule.forRoot(),
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST || 'localhost',
      port: parseInt(process.env.DB_PORT) || 5432,
      username: process.env.DB_USERNAME || 'postgres',
      password: process.env.DB_PASSWORD || 'senha123',
      database: process.env.DB_DATABASE || 'meubanco',
      autoLoadEntities: true,
      synchronize: true, // Apenas para desenvolvimento
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

**src/app.controller.ts:**
```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get('health')
  async getHealth() {
    return await this.appService.getHealth();
  }
}
```

**src/app.service.ts:**
```typescript
import { Injectable } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

@Injectable()
export class AppService {
  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
  ) {}

  getHello(): string {
    return 'Olá Docker! API NestJS com PostgreSQL 🐳🐘';
  }

  async getHealth() {
    try {
      await this.dataSource.query('SELECT 1');
      return {
        status: 'OK',
        message: 'API e banco funcionando perfeitamente!',
        timestamp: new Date().toISOString(),
        database: 'PostgreSQL conectado',
        environment: 'Docker Container'
      };
    } catch (error) {
      return {
        status: 'ERROR',
        message: 'Erro na conexão com o banco',
        error: error.message
      };
    }
  }
}
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # API NestJS
  api:
    build: .
    container_name: api-nestjs
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USERNAME=postgres
      - DB_PASSWORD=senha123
      - DB_DATABASE=meubanco
    depends_on:
      - postgres
    networks:
      - app-network

  # Banco PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=senha123
      - POSTGRES_DB=meubanco
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
```

**Dockerfile:**
```dockerfile
FROM node:18-alpine

# Instalar NestJS CLI
RUN npm install -g @nestjs/cli

WORKDIR /app

# Copiar e instalar dependências
COPY package*.json ./
RUN npm ci --only=production

# Copiar código
COPY . .

# Compilar aplicação
RUN npm run build

EXPOSE 3000

CMD ["npm", "run", "start:prod"]
```

**Execução:**
```bash
# Subir todos os serviços
docker-compose up -d

# Ver logs da API
docker-compose logs api

# Ver logs do banco
docker-compose logs postgres

# Testar a API
curl http://localhost:3000
curl http://localhost:3000/health

# Parar serviços
docker-compose down
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

## Melhores Práticas

### Dockerfile
- **Use imagens oficiais**: `node:18-alpine` em vez de `ubuntu + instalar node`
- **Minimize camadas**: Combine comandos `RUN` com `&&`
- **Use .dockerignore**: Exclua `node_modules`, `.git`, etc.
- **Ordene instruções**: Coloque mudanças frequentes por último
- **Use usuário não-root**: `USER node` para segurança

**Exemplo de Dockerfile otimizado para NestJS:**
```dockerfile
FROM node:18-alpine

# Instalar NestJS CLI
RUN npm install -g @nestjs/cli

# Criar usuário não-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

WORKDIR /app

# Copiar e instalar dependências primeiro (cache)
COPY package*.json ./
RUN npm ci --only=production

# Copiar código depois
COPY --chown=nestjs:nodejs . .

# Compilar aplicação
RUN npm run build

USER nestjs

EXPOSE 3000
CMD ["npm", "run", "start:prod"]
```

### Segurança
- **Imagens atualizadas**: Use `docker scan` para vulnerabilidades
- **Sem secrets**: Use variáveis de ambiente ou secrets
- **Multi-stage builds**: Reduz tamanho da imagem final
- **Privilégios mínimos**: Não use `--privileged`

### Performance
- **Cache de build**: Ordene instruções por frequência de mudança
- **Imagens pequenas**: Prefira Alpine Linux
- **Volumes**: Para dados persistentes
- **Health checks**: Monitore saúde dos containers

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
- Desenvolvimento de APIs modernas (NestJS, Express, Fastify)
- Microserviços
- CI/CD pipelines
- Aplicações stateless
- Escalabilidade horizontal é importante
- Portabilidade entre ambientes é crítica
- Desenvolvimento com TypeScript/JavaScript

### ❌ Evite Docker quando:
- Aplicações com GUI complexa
- Performance crítica (jogos, HPC)
- Aplicações que precisam de kernel customizado
- Sistemas legados muito complexos

## Referências

Este conteúdo é baseado na documentação oficial do [Docker](https://docs.docker.com/), melhores práticas da comunidade e experiência prática em desenvolvimento.
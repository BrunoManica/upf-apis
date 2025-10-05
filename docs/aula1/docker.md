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
  - Ex: `nginx`, `postgres`, `node`

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

Este conteúdo é baseado na documentação oficial do [Docker](https://docs.docker.com/).
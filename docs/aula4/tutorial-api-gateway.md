# Tutorial: Construindo um API Gateway com NestJS

## Introdução

Este tutorial demonstra como construir um **API Gateway** completo usando NestJS. Vamos criar:

1. **API Gateway** → ponto único de entrada com rate limiting
2. **Microsserviço de Usuários** → serviço que gerencia usuários
3. **Microsserviço de Produtos** → serviço que gerencia produtos

O API Gateway será responsável por:
- Rotear requisições para os microsserviços corretos
- Aplicar rate limiting (limitação de taxa)
- Centralizar o acesso aos serviços


## Pré-requisitos

- Node.js 18 ou superior
- npm ou yarn
- Conhecimento básico de TypeScript e NestJS
- Conhecimento básico de APIs REST

## Passo 1: Estrutura do Projeto

Vamos criar 3 projetos NestJS separados em um único diretório:

```bash
# Criar diretório principal
mkdir gateway-tutorial
cd gateway-tutorial

# Criar os 3 projetos
nest new gateway-api
nest new microservice-users
nest new microservice-products
```

**Estrutura final:**
```
gateway-tutorial/
├── gateway-api/
├── microservice-users/
└── microservice-products/
```

## Passo 2: Configurar Microsserviço de Usuários

### 2.1 Criar Resource de Usuários

```bash
cd microservice-users
nest g resource users
```

Quando perguntado:
- **What transport layer do you use?** → Escolha `REST API`
- **Would you like to generate CRUD entry points?** → Digite `N` (não precisamos de todos os endpoints CRUD)

### 2.2 Implementar Service de Usuários

Editar `src/users/users.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  findAll() {
    return [
      { id: 1, name: 'Ana', email: 'ana@email.com' },
      { id: 2, name: 'Lucas', email: 'lucas@email.com' },
      { id: 3, name: 'Maria', email: 'maria@email.com' },
    ];
  }
}
```

### 2.3 Implementar Controller de Usuários

Editar `src/users/users.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }
}
```

### 2.4 Configurar Porta

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Habilitar CORS para aceitar requisições do gateway
  app.enableCors();
  
  const port = process.env.PORT || 3001;
  await app.listen(port);
  
  console.log(`👤 Microsserviço de Usuários rodando na porta ${port}`);
}
bootstrap();
```

## Passo 3: Configurar Microsserviço de Produtos

### 3.1 Criar Resource de Produtos

```bash
cd ../microservice-products
nest g resource products
```

Quando perguntado:
- **What transport layer do you use?** → Escolha `REST API`
- **Would you like to generate CRUD entry points?** → Digite `N` (não precisamos de todos os endpoints CRUD)

### 3.2 Implementar Service de Produtos

Editar `src/products/products.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class ProductsService {
  findAll() {
    return [
      { id: 1, name: 'Notebook', price: 2500.00 },
      { id: 2, name: 'Mouse', price: 50.00 },
      { id: 3, name: 'Teclado', price: 150.00 },
    ];
  }
}
```

### 3.3 Implementar Controller de Produtos

Editar `src/products/products.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';
import { ProductsService } from './products.service';

@Controller('products')
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  findAll() {
    return this.productsService.findAll();
  }
}
```

### 3.4 Configurar Porta

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Habilitar CORS para aceitar requisições do gateway
  app.enableCors();
  
  const port = process.env.PORT || 3002;
  await app.listen(port);
  
  console.log(`📦 Microsserviço de Produtos rodando na porta ${port}`);
}
bootstrap();
```

## Passo 4: Configurar API Gateway

### 4.1 Instalar Dependências

```bash
cd ../gateway-api
npm install @nestjs/throttler @nestjs/config axios
```

**Dependências:**
- `@nestjs/throttler` → rate limiting
- `@nestjs/config` → gerenciamento de configurações
- `axios` → fazer requisições HTTP para os microsserviços

### 4.2 Criar Resource de Proxy

```bash
nest g resource proxy
```

Quando perguntado:
- **What transport layer do you use?** → Escolha `REST API`
- **Would you like to generate CRUD entry points?** → Digite `N` (não precisamos de todos os endpoints CRUD)

**Vantagem:** O comando `nest g resource` cria automaticamente:
- Módulo (`proxy.module.ts`)
- Service (`proxy.service.ts`)
- Controller (`proxy.controller.ts`)
- DTOs (se necessário)

É mais rápido e organizado que criar cada arquivo separadamente!

### 4.3 Implementar Proxy Service

Editar `src/proxy/proxy.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class ProxyService {
  // URL dos microsserviços (vindas das variáveis de ambiente)
  private readonly productsApi = process.env.PRODUCTS_API || 'http://localhost:3002';
  private readonly usersApi = process.env.USERS_API || 'http://localhost:3001';

  /**
   * Busca produtos do microsserviço de produtos
   */
  async getProducts() {
    const { data } = await axios.get(`${this.productsApi}/products`);
    return data;
  }

  /**
   * Busca usuários do microsserviço de usuários
   */
  async getUsers() {
    const { data } = await axios.get(`${this.usersApi}/users`);
    return data;
  }
}
```

**Explicação:**

- O `ProxyService` atua como intermediário
- Usa `axios` para fazer requisições HTTP aos microsserviços
- URLs vêm de variáveis de ambiente (configuráveis)
- Cada método encapsula a lógica de comunicação com um serviço específico

### 4.4 Implementar Proxy Controller

Editar `src/proxy/proxy.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';
import { SkipThrottle } from '@nestjs/throttler';
import { ProxyService } from './proxy.service';

@SkipThrottle() // Desabilita todos os throttlers por padrão
@Controller('proxy')
export class ProxyController {
  constructor(private readonly proxyService: ProxyService) {}

  /**
   * Endpoint para buscar usuários
   * Ativa apenas o throttler 'users' (5 requisições por minuto)
   */
  @SkipThrottle({ users: false })
  @Get('users')
  getUsers() {
    return this.proxyService.getUsers();
  }

  /**
   * Endpoint para buscar produtos
   * Ativa apenas o throttler 'products' (30 requisições por minuto)
   */
  @SkipThrottle({ products: false })
  @Get('products')
  getProducts() {
    return this.proxyService.getProducts();
  }
}
```

**Explicação:**

- `@SkipThrottle()` desabilita rate limiting por padrão
- `@SkipThrottle({ users: false })` ativa o throttler específico chamado 'users'
- Cada endpoint tem seu próprio limite de requisições

### 4.5 Configurar Rate Limiting

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { APP_FILTER, APP_GUARD } from '@nestjs/core';
import { ThrottlerGuard, ThrottlerModule } from '@nestjs/throttler';
import { ThrottlerExceptionFilter } from './filters/throttle-exception.filter';
import { ProxyModule } from './proxy/proxy.module';

@Module({
  providers: [
    // Filtro global para tratar erros de rate limiting
    {
      provide: APP_FILTER,
      useClass: ThrottlerExceptionFilter,
    },
    // Guard global para aplicar rate limiting
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
  imports: [
    // Módulo de configuração (carrega variáveis de ambiente)
    ConfigModule.forRoot({ isGlobal: true }),
    
    // Módulo de rate limiting com configurações
    ThrottlerModule.forRoot([
      {
        name: 'users',
        ttl: 60000,        // 60 segundos (1 minuto)
        limit: 5,          // 5 requisições por minuto
      },
      {
        name: 'products',
        ttl: 60000,        // 60 segundos (1 minuto)
        limit: 30,         // 30 requisições por minuto
      },
    ]),
    
    ProxyModule,
  ],
})
export class AppModule {}
```

**Explicação:**

- `ThrottlerModule.forRoot()` configura múltiplos throttlers
- Cada throttler tem:
  - `name`: identificador único
  - `ttl`: tempo em milissegundos (time-to-live)
  - `limit`: número máximo de requisições no período

### 4.6 Criar Filtro de Exceção para Rate Limiting

```bash
nest g filter filters/throttle-exception
```

Editar `src/filters/throttle-exception.filter.ts`:

```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpStatus,
} from '@nestjs/common';
import { ThrottlerException } from '@nestjs/throttler';
import { Response } from 'express';

@Catch(ThrottlerException)
export class ThrottlerExceptionFilter implements ExceptionFilter {
  catch(exception: ThrottlerException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest();

    response.status(HttpStatus.TOO_MANY_REQUESTS).json({
      statusCode: HttpStatus.TOO_MANY_REQUESTS,
      timestamp: new Date().toISOString(),
      path: request.url,
      message:
        'Você excedeu o limite de requisições. Tente novamente em alguns instantes.',
      error: 'Too Many Requests',
    });
  }
}
```

**Explicação:**

- Intercepta erros de rate limiting
- Retorna uma resposta amigável (HTTP 429)
- Inclui informações úteis: timestamp, path, mensagem

### 4.7 Configurar Porta do Gateway

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Habilitar CORS
  app.enableCors();
  
  const port = process.env.PORT || 3000;
  await app.listen(port);
  
  console.log(`🚪 API Gateway rodando na porta ${port}`);
  console.log(`📡 Endpoints disponíveis:`);
  console.log(`   - http://localhost:${port}/proxy/users`);
  console.log(`   - http://localhost:${port}/proxy/products`);
}
bootstrap();
```

## Passo 5: Configurar Variáveis de Ambiente (Opcional)

Criar arquivo `.env` no `gateway-api` (opcional, pois já temos valores padrão):

```bash
# gateway-api/.env
PORT=3000
USERS_API=http://localhost:3001
PRODUCTS_API=http://localhost:3002
THROTTLE_TTL_USERS=60000
THROTTLE_LIMIT_USERS=5
THROTTLE_TTL_PRODUCTS=60000
THROTTLE_LIMIT_PRODUCTS=30
```

## Passo 6: Testar o Sistema

### 6.1 Iniciar os Serviços

**Terminal 1 - Microsserviço de Usuários:**
```bash
cd microservice-users
npm run start:dev
```

**Terminal 2 - Microsserviço de Produtos:**
```bash
cd microservice-products
npm run start:dev
```

**Terminal 3 - API Gateway:**
```bash
cd gateway-api
npm run start:dev
```

### 6.2 Testar Endpoints Diretos (Sem Gateway)

**Testar Microsserviço de Usuários diretamente:**
```bash
curl http://localhost:3001/users
```

**Resposta esperada:**
```json
[
  { "id": 1, "name": "Ana", "email": "ana@email.com" },
  { "id": 2, "name": "Lucas", "email": "lucas@email.com" },
  { "id": 3, "name": "Maria", "email": "maria@email.com" }
]
```

**Testar Microsserviço de Produtos diretamente:**
```bash
curl http://localhost:3002/products
```

**Resposta esperada:**
```json
[
  { "id": 1, "name": "Notebook", "price": 2500.00 },
  { "id": 2, "name": "Mouse", "price": 50.00 },
  { "id": 3, "name": "Teclado", "price": 150.00 }
]
```

### 6.3 Testar através do API Gateway

**Testar Usuários via Gateway:**
```bash
curl http://localhost:3000/proxy/users
```

**O que acontece:**
1. Cliente faz requisição para o gateway
2. Gateway roteia para `http://localhost:3001/users`
3. Gateway retorna a resposta ao cliente

**Testar Produtos via Gateway:**
```bash
curl http://localhost:3000/proxy/products
```

### 6.4 Testar Rate Limiting

**Testar Limite de Usuários (5 requisições/minuto):**

Execute 6 requisições rápidas:

```bash
# Requisições 1-5: sucesso
curl http://localhost:3000/proxy/users
curl http://localhost:3000/proxy/users
curl http://localhost:3000/proxy/users
curl http://localhost:3000/proxy/users
curl http://localhost:3000/proxy/users

# Requisição 6: deve retornar erro 429
curl http://localhost:3000/proxy/users
```

**Resposta esperada na 6ª requisição:**
```json
{
  "statusCode": 429,
  "timestamp": "2024-01-01T10:00:00.000Z",
  "path": "/proxy/users",
  "message": "Você excedeu o limite de requisições. Tente novamente em alguns instantes.",
  "error": "Too Many Requests"
}
```

**Testar Limite de Produtos (30 requisições/minuto):**

Execute 31 requisições rápidas para verificar o limite:

```bash
# Requisições 1-30: sucesso
for i in {1..30}; do
  curl http://localhost:3000/proxy/products
done

# Requisição 31: deve retornar erro 429
curl http://localhost:3000/proxy/products
```

**Observação:** Os limites são por endpoint, não compartilhados. Você pode fazer:
- 5 requisições para `/proxy/users`
- 30 requisições para `/proxy/products`
- Simultaneamente!

## Passo 7: Entendendo o Fluxo de Requisições

### Fluxo Completo: Cliente → Gateway → Microsserviço

```
1. Cliente faz requisição:
   GET http://localhost:3000/proxy/users
   │
   ▼
2. API Gateway recebe:
   - Endpoint: /proxy/users
   - Throttler 'users' verifica limite (5/min)
   │
   ▼
3. Gateway verifica rate limiting:
   - Se dentro do limite: continua
   - Se excedido: retorna 429 (Too Many Requests)
   │
   ▼
4. ProxyService.getUsers() é chamado:
   - Faz requisição: GET http://localhost:3001/users
   │
   ▼
5. Microsserviço de Usuários responde:
   - Retorna lista de usuários
   │
   ▼
6. Gateway retorna resposta ao cliente:
   - Cliente recebe os dados dos usuários
```

### Vantagens do API Gateway

**Sem Gateway:**
```
Cliente precisa conhecer:
- http://localhost:3001/users  (microsserviço de usuários)
- http://localhost:3002/products  (microsserviço de produtos)

Problemas:
- Cliente precisa saber portas de cada serviço
- Não há rate limiting centralizado
- Dificulta mudanças futuras
```

**Com Gateway:**
```
Cliente só conhece:
- http://localhost:3000/proxy/*  (API Gateway)

Vantagens:
- Ponto único de entrada
- Rate limiting centralizado
- Fácil de adicionar novos serviços
- Segurança centralizada (pode adicionar autenticação)
```

## Passo 8: Adicionar Logging (Opcional)

Para entender melhor o fluxo, podemos adicionar logs no gateway:

Editar `src/proxy/proxy.service.ts`:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class ProxyService {
  private readonly logger = new Logger(ProxyService.name);

  private readonly productsApi = process.env.PRODUCTS_API || 'http://localhost:3002';
  private readonly usersApi = process.env.USERS_API || 'http://localhost:3001';

  async getProducts() {
    this.logger.log('Buscando produtos do microsserviço de produtos...');
    const { data } = await axios.get(`${this.productsApi}/products`);
    this.logger.log(`Produtos retornados: ${data.length} itens`);
    return data;
  }

  async getUsers() {
    this.logger.log('Buscando usuários do microsserviço de usuários...');
    const { data } = await axios.get(`${this.usersApi}/users`);
    this.logger.log(`Usuários retornados: ${data.length} itens`);
    return data;
  }
}
```

**Ao fazer uma requisição, você verá no console:**
```
[ProxyService] Buscando usuários do microsserviço de usuários...
[ProxyService] Usuários retornados: 3 itens
```

## Passo 9: Adicionar Tratamento de Erros (Opcional)

Melhorar o `ProxyService` para tratar erros:

```typescript
import { Injectable, Logger, HttpException, HttpStatus } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class ProxyService {
  private readonly logger = new Logger(ProxyService.name);

  private readonly productsApi = process.env.PRODUCTS_API || 'http://localhost:3002';
  private readonly usersApi = process.env.USERS_API || 'http://localhost:3001';

  async getProducts() {
    try {
      this.logger.log('Buscando produtos...');
      const { data } = await axios.get(`${this.productsApi}/products`);
      return data;
    } catch (error) {
      this.logger.error('Erro ao buscar produtos', error);
      throw new HttpException(
        'Erro ao buscar produtos. Serviço temporariamente indisponível.',
        HttpStatus.SERVICE_UNAVAILABLE,
      );
    }
  }

  async getUsers() {
    try {
      this.logger.log('Buscando usuários...');
      const { data } = await axios.get(`${this.usersApi}/users`);
      return data;
    } catch (error) {
      this.logger.error('Erro ao buscar usuários', error);
      throw new HttpException(
        'Erro ao buscar usuários. Serviço temporariamente indisponível.',
        HttpStatus.SERVICE_UNAVAILABLE,
      );
    }
  }
}
```

**Se o microsserviço estiver offline, o gateway retornará:**
```json
{
  "statusCode": 503,
  "message": "Erro ao buscar produtos. Serviço temporariamente indisponível."
}
```

## Resumo do Tutorial

### O que construímos:

1. **Microsserviço de Usuários** (porta 3001)
   - Endpoint: `GET /users`
   - Retorna lista de usuários

2. **Microsserviço de Produtos** (porta 3002)
   - Endpoint: `GET /products`
   - Retorna lista de produtos

3. **API Gateway** (porta 3000)
   - Endpoint: `GET /proxy/users` → roteia para microsserviço de usuários
   - Endpoint: `GET /proxy/products` → roteia para microsserviço de produtos
   - Rate limiting:
     - Usuários: 5 requisições/minuto
     - Produtos: 30 requisições/minuto

### Conceitos Aprendidos:

✅ **API Gateway como ponto único de entrada**
✅ **Rate Limiting por endpoint**
✅ **Roteamento de requisições para microsserviços**
✅ **Tratamento de erros centralizado**
✅ **Arquitetura de microsserviços com gateway**

### Próximos Passos (Extensões Possíveis):

1. **Autenticação**: Adicionar JWT ou OAuth
2. **Cache**: Implementar cache de respostas
3. **Load Balancing**: Distribuir carga entre múltiplas instâncias
4. **Monitoring**: Adicionar métricas e observabilidade
5. **Circuit Breaker**: Proteger contra falhas em cascata
6. **API Versioning**: Suportar múltiplas versões de API

## Comandos Úteis

### Iniciar Todos os Serviços

**Script simples (criar `start-all.sh`):**
```bash
#!/bin/bash

# Iniciar microsserviços em background
cd microservice-users && npm run start:dev &
cd ../microservice-products && npm run start:dev &
cd ../gateway-api && npm run start:dev
```

**Ou usar Docker Compose** (tutorial avançado):

```yaml
version: '3.8'
services:
  users:
    build: ./microservice-users
    ports:
      - "3001:3001"
  
  products:
    build: ./microservice-products
    ports:
      - "3002:3002"
  
  gateway:
    build: ./gateway-api
    ports:
      - "3000:3000"
    environment:
      - USERS_API=http://users:3001
      - PRODUCTS_API=http://products:3002
    depends_on:
      - users
      - products
```

## Conclusão

Este tutorial demonstrou como construir um **API Gateway funcional** usando NestJS. O gateway centraliza o acesso aos microsserviços, aplica rate limiting e facilita o gerenciamento da arquitetura.

**Principais Benefícios:**

- **Simplificação**: Cliente só conhece o gateway
- **Segurança**: Rate limiting centralizado
- **Manutenibilidade**: Fácil adicionar novos serviços
- **Observabilidade**: Logs e métricas centralizados

O API Gateway é uma peça fundamental em arquiteturas modernas, especialmente em sistemas com múltiplos microsserviços!


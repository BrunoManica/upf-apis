# Tutorial: Implementando Observabilidade em NestJS

## Introdução

Este tutorial demonstra como implementar **observabilidade profissional** em uma aplicação NestJS. Vamos criar:

1. **Logger Customizado** → substitui `console.log` por logging estruturado
2. **Logging Interceptor** → mede tempo de requisições e registra automaticamente
3. **Health Check** → verifica saúde da aplicação e dependências
4. **Métricas Prometheus** → expõe métricas para coleta
5. **Tracing OpenTelemetry** → rastreia requisições através do sistema

**Objetivo:** Aprender a implementar observabilidade de forma profissional, sem usar `console.log`, seguindo boas práticas do NestJS.

## Pré-requisitos

- Node.js 18 ou superior
- npm ou yarn
- Conhecimento básico de TypeScript e NestJS
- NestJS CLI instalado globalmente (`npm install -g @nestjs/cli`)

## Passo 1: Criar Novo Projeto NestJS

### 1.1 Criar Projeto

```bash
# Criar novo projeto NestJS
nest new observability-api

# Quando perguntado sobre package manager, escolha npm ou yarn
# Nome do projeto: observability-api
```

### 1.2 Navegar para o Projeto

```bash
cd observability-api
```

### 1.3 Instalar Dependências de Observabilidade

```bash
npm install @nestjs/terminus prom-client
```

**Nota:** O pacote `prom-client` já inclui tipos TypeScript, então não é necessário instalar `@types/prom-client`.

**Dependências:**

- `@nestjs/terminus` → Health checks
- `prom-client` → Métricas Prometheus

**Observação:** Para OpenTelemetry, vamos usar uma abordagem simplificada primeiro. Em produção, você pode instalar `@nestjs/otel` para tracing completo.

### 1.4 Criar Estrutura de Pastas

```bash
# Criar pastas para organização
mkdir -p src/common/logger
mkdir -p src/common/interceptors
mkdir -p src/health
mkdir -p src/metrics
```

**Estrutura final do projeto:**

```
observability-api/
├── src/
│   ├── common/
│   │   ├── logger/
│   │   │   └── my-logger.service.ts
│   │   └── interceptors/
│   │       └── logging.interceptor.ts
│   ├── health/
│   │   ├── health.controller.ts
│   │   └── health.module.ts
│   ├── metrics/
│   │   ├── metrics.controller.ts
│   │   ├── metrics.service.ts
│   │   └── metrics.module.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
└── package.json
```

## Passo 2: Criar Logger Customizado

### 2.1 Por que um Logger Customizado?

**Problema com `console.log`:**

- Não tem níveis de log (info, error, warn, debug)
- Não tem contexto (não sabemos de onde veio o log)
- Difícil filtrar e buscar logs
- Não é estruturado (difícil processar automaticamente)

**Solução: Logger Customizado**

- Níveis de log claros (log, error, warn, debug)
- Contexto automático (nome da classe/módulo)
- Estruturado (pode ser processado por ferramentas)
- Pode ser estendido (adicionar formatação, envio para serviços externos)

### 2.2 Criar MyLoggerService

Criar arquivo `src/common/logger/my-logger.service.ts`:

```bash
# Criar arquivo
touch src/common/logger/my-logger.service.ts
```

Editar o arquivo:

```typescript
import { Injectable, ConsoleLogger } from '@nestjs/common';

/**
 * Logger Customizado
 * 
 * Por que criar um logger customizado?
 * - Substitui console.log por logging estruturado
 * - Adiciona contexto (nome da classe/módulo)
 * - Permite extensão futura (envio para serviços externos, formatação JSON)
 * - Segue padrões do NestJS
 * 
 * Como usar:
 * - Injetar no construtor: constructor(private logger: MyLoggerService) {}
 * - Usar: this.logger.log('Mensagem', 'Contexto')
 */
@Injectable()
export class MyLoggerService extends ConsoleLogger {
  /**
   * Log de informação geral
   * Use para registrar eventos normais do sistema
   */
  log(message: string, context?: string) {
    // Formatação melhorada com timestamp e contexto
    super.log(message, context || this.context || 'Application');
  }

  /**
   * Log de erros
   * Use para registrar erros e exceções
   */
  error(message: string, stack?: string, context?: string) {
    // Stack trace incluído para debugging
    super.error(message, stack, context || this.context || 'Application');
  }

  /**
   * Log de avisos
   * Use para situações que merecem atenção mas não são erros
   */
  warn(message: string, context?: string) {
    super.warn(message, context || this.context || 'Application');
  }

  /**
   * Log de debug
   * Use para informações detalhadas durante desenvolvimento
   */
  debug(message: string, context?: string) {
    super.debug(message, context || this.context || 'Application');
  }

  /**
   * Log de informações verbosas
   * Use para informações muito detalhadas
   */
  verbose(message: string, context?: string) {
    super.verbose(message, context || this.context || 'Application');
  }
}
```

**Explicação:**

- `extends ConsoleLogger`: Herda funcionalidades do logger padrão do NestJS
- `@Injectable()`: Permite injeção de dependência
- Métodos sobrescritos: Adicionam formatação e contexto padrão
- Contexto automático: Se não fornecido, usa o contexto da classe

## Passo 3: Criar Logging Interceptor

### 3.1 Por que um Interceptor?

**Problema:** Sem interceptor, você precisa adicionar logs manualmente em cada endpoint:

```typescript
//  Ruim: Logging manual em cada método
@Get()
findAll() {
  console.log('Iniciando busca de pagamentos');
  const start = Date.now();
  const result = this.service.findAll();
  const time = Date.now() - start;
  console.log(`Busca concluída em ${time}ms`);
  return result;
}
```

**Solução: Interceptor Global**

- Logging automático de todas as requisições
- Medição de tempo de resposta
- Contexto automático (método HTTP, URL)
- Não precisa modificar cada controller

### 3.2 Criar LoggingInterceptor

Criar arquivo `src/common/interceptors/logging.interceptor.ts`:

```bash
# Criar arquivo
touch src/common/interceptors/logging.interceptor.ts
```

Editar o arquivo:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { MyLoggerService } from '../logger/my-logger.service';

/**
 * Logging Interceptor
 * 
 * O que faz:
 * - Registra automaticamente todas as requisições HTTP
 * - Mede o tempo de resposta
 * - Adiciona contexto (método, URL, status code)
 * 
 * Por que usar:
 * - Não precisa adicionar logs manualmente em cada endpoint
 * - Visibilidade automática de todas as requisições
 * - Identifica endpoints lentos facilmente
 * 
 * Como funciona:
 * 1. Intercepta requisição antes de processar
 * 2. Registra início da requisição
 * 3. Processa requisição
 * 4. Mede tempo decorrido
 * 5. Registra fim da requisição com tempo
 */
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private readonly logger: MyLoggerService) {
    // Define contexto do logger
    this.logger.setContext('HTTP');
  }

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // Obtém informações da requisição HTTP
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const now = Date.now();

    // Registra início da requisição
    this.logger.log(`→ ${method} ${url}`, 'HTTP');

    // Processa requisição e mede tempo
    return next.handle().pipe(
      tap({
        // Quando requisição é bem-sucedida
        next: () => {
          const response = context.switchToHttp().getResponse();
          const { statusCode } = response;
          const time = Date.now() - now;
          
          this.logger.log(
            `← ${method} ${url} ${statusCode} - ${time}ms`,
            'HTTP',
          );
        },
        // Quando requisição falha
        error: (error) => {
          const time = Date.now() - now;
          const statusCode = error.status || 500;
          
          this.logger.error(
            `${method} ${url} ${statusCode} - ${time}ms - ${error.message}`,
            error.stack,
            'HTTP',
          );
        },
      }),
    );
  }
}
```

**Explicação:**

- `NestInterceptor`: Interface do NestJS para interceptors
- `intercept()`: Método chamado para cada requisição
- `next.handle()`: Continua processamento da requisição
- `tap()`: Operador RxJS que executa código sem modificar o fluxo
- Medição de tempo: `Date.now()` antes e depois

**Exemplo de saída:**
```
[HTTP] → GET /api/payments
[HTTP] ← GET /api/payments 200 - 45ms
```

### 3.3 Registrar Logger e Interceptor no AppModule

Agora que criamos tanto o Logger quanto o Interceptor, vamos registrá-los no `AppModule`:

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLoggerService } from './common/logger/my-logger.service';
import { LoggingInterceptor } from './common/interceptors/logging.interceptor';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [
    AppService,
    // Registrar logger como provider global
    {
      provide: MyLoggerService,
      useValue: new MyLoggerService(),
    },
    // Registrar interceptor global
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

**Explicação:**

- `MyLoggerService`: Provider global para usar o logger em qualquer lugar da aplicação
- `APP_INTERCEPTOR`: Token especial do NestJS para registrar interceptors globalmente
- `LoggingInterceptor`: Interceptor que será aplicado automaticamente a todas as requisições HTTP

## Passo 4: Implementar Health Check

### 4.1 Por que Health Check?

**Problema:** Como saber se a aplicação está funcionando corretamente?

**Solução: Health Check**

- Endpoint `/health` que verifica saúde da aplicação
- Verifica dependências (banco de dados, APIs externas)
- Usado por orquestradores (Kubernetes, Docker Swarm)
- Monitoramento automático

### 4.2 Criar Health Module

```bash
# Criar módulo e controller usando NestJS CLI
nest g module health
nest g controller health
```

**Ou criar manualmente:**

```bash
# Criar arquivos manualmente
touch src/health/health.module.ts
touch src/health/health.controller.ts
```

### 4.3 Implementar Health Controller

Editar `src/health/health.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HealthCheck,
  MemoryHealthIndicator,
  DiskHealthIndicator,
} from '@nestjs/terminus';
import { MyLoggerService } from '../common/logger/my-logger.service';

/**
 * Health Check Controller
 * 
 * O que faz:
 * - Fornece endpoint /health para verificar saúde da aplicação
 * - Verifica memória, disco, banco de dados
 * - Usado por orquestradores (Kubernetes) para verificar se app está saudável
 * 
 * Endpoints:
 * - GET /health → Verifica saúde geral
 * - GET /health/live → Verifica se aplicação está rodando (liveness)
 * - GET /health/ready → Verifica se aplicação está pronta (readiness)
 */
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator,
    private logger: MyLoggerService,
  ) {
    this.logger.setContext('HealthCheck');
  }

  /**
   * Health check completo
   * Verifica memória, disco e outras dependências
   */
  @Get()
  @HealthCheck()
  check() {
    this.logger.log('Health check executado', 'HealthCheck');
    
    return this.health.check([
      // Verifica uso de memória (alerta se acima de 300MB)
      () => this.memory.checkHeap('memory_heap', 300 * 1024 * 1024),
      
      // Verifica uso de memória RSS (alerta se acima de 300MB)
      () => this.memory.checkRSS('memory_rss', 300 * 1024 * 1024),
      
      // Verifica espaço em disco (alerta se acima de 90%)
      () =>
        this.disk.checkStorage('storage', {
          path: '/',
          thresholdPercent: 0.9,
        }),
    ]);
  }

  /**
   * Liveness probe
   * Verifica se aplicação está viva (rodando)
   * Kubernetes usa isso para reiniciar containers
   */
  @Get('live')
  @HealthCheck()
  liveness() {
    return this.health.check([
      () => ({
        app: {
          status: 'up',
        },
      }),
    ]);
  }

  /**
   * Readiness probe
   * Verifica se aplicação está pronta para receber tráfego
   * Kubernetes usa isso para rotear tráfego
   */
  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([
      () => ({
        app: {
          status: 'up',
        },
      }),
    ]);
  }
}
```

### 4.4 Configurar Health Module

Editar `src/health/health.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HealthController } from './health.controller';
import { MyLoggerService } from '../common/logger/my-logger.service';

@Module({
  imports: [TerminusModule],
  controllers: [HealthController],
  providers: [MyLoggerService],
})
export class HealthModule {}
```

### 4.5 Registrar Health Module

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLoggerService } from './common/logger/my-logger.service';
import { LoggingInterceptor } from './common/interceptors/logging.interceptor';
import { HealthModule } from './health/health.module';

@Module({
  imports: [
    HealthModule, // Adicionar health module
  ],
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: MyLoggerService,
      useValue: new MyLoggerService(),
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

### 4.6 Testar Health Check

```bash
# Iniciar aplicação
npm run start:dev

# Testar health check
curl http://localhost:3001/health
```

**Resposta esperada:**
```json
{
  "status": "ok",
  "info": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" },
    "storage": { "status": "up" }
  },
  "error": {},
  "details": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" },
    "storage": { "status": "up" }
  }
}
```

## Passo 5: Implementar Métricas Prometheus

### 5.1 Por que Prometheus?

**Problema:** Como coletar métricas da aplicação?

**Solução: Prometheus**

- Padrão de mercado para métricas
- Formato exposto em `/metrics`
- Coletado por Prometheus server
- Visualizado em Grafana

### 5.2 Criar Metrics Module

```bash
# Criar módulo e controller usando NestJS CLI
nest g module metrics
nest g controller metrics
nest g service metrics
```

**Ou criar manualmente:**

```bash
# Criar arquivos manualmente
touch src/metrics/metrics.module.ts
touch src/metrics/metrics.controller.ts
touch src/metrics/metrics.service.ts
```

### 5.3 Implementar Metrics Service

Criar arquivo `src/metrics/metrics.service.ts`:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { Registry, Counter, Histogram, collectDefaultMetrics } from 'prom-client';
import { MyLoggerService } from '../common/logger/my-logger.service';

/**
 * Metrics Service
 * 
 * O que faz:
 * - Expõe métricas no formato Prometheus
 * - Coleta métricas padrão (CPU, memória, etc)
 * - Permite criar métricas customizadas
 * 
 * Métricas expostas:
 * - http_requests_total: Total de requisições HTTP
 * - http_request_duration_seconds: Duração das requisições
 * - Métricas padrão do Node.js (CPU, memória, etc)
 */
@Injectable()
export class MetricsService implements OnModuleInit {
  private readonly registry: Registry;
  public readonly httpRequestCounter: Counter;
  public readonly httpRequestDuration: Histogram;

  constructor(private readonly logger: MyLoggerService) {
    this.logger.setContext('Metrics');
    
    // Criar registry para métricas
    this.registry = new Registry();
    
    // Coletar métricas padrão do Node.js
    collectDefaultMetrics({ register: this.registry });

    // Contador de requisições HTTP
    this.httpRequestCounter = new Counter({
      name: 'http_requests_total',
      help: 'Total de requisições HTTP',
      labelNames: ['method', 'route', 'status'],
      registers: [this.registry],
    });

    // Histograma de duração de requisições
    this.httpRequestDuration = new Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duração das requisições HTTP em segundos',
      labelNames: ['method', 'route'],
      buckets: [0.1, 0.5, 1, 2, 5], // Buckets para percentis
      registers: [this.registry],
    });
  }

  onModuleInit() {
    this.logger.log('Métricas Prometheus inicializadas', 'Metrics');
  }

  /**
   * Retorna métricas no formato Prometheus
   */
  async getMetrics(): Promise<string> {
    return this.registry.metrics();
  }
}
```

### 5.4 Implementar Metrics Controller

Editar `src/metrics/metrics.controller.ts`:

```typescript
import { Controller, Get, Header } from '@nestjs/common';
import { MetricsService } from './metrics.service';

/**
 * Metrics Controller
 * 
 * Endpoint: GET /metrics
 * 
 * Retorna métricas no formato Prometheus
 * Prometheus server coleta essas métricas periodicamente
 */
@Controller('metrics')
export class MetricsController {
  constructor(private readonly metricsService: MetricsService) {}

  @Get()
  @Header('Content-Type', 'text/plain')
  async getMetrics() {
    return this.metricsService.getMetrics();
  }
}
```

### 5.5 Configurar Metrics Module

Editar `src/metrics/metrics.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { MetricsController } from './metrics.controller';
import { MetricsService } from './metrics.service';
import { MyLoggerService } from '../common/logger/my-logger.service';

@Module({
  controllers: [MetricsController],
  providers: [MetricsService, MyLoggerService],
  exports: [MetricsService], // Exportar para usar em outros módulos
})
export class MetricsModule {}
```

### 5.6 Integrar Métricas no Interceptor

Atualizar `src/common/interceptors/logging.interceptor.ts`:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
  Optional,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { MyLoggerService } from '../logger/my-logger.service';
import { MetricsService } from '../../metrics/metrics.service';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(
    private readonly logger: MyLoggerService,
    @Optional() @Inject(MetricsService) private readonly metrics?: MetricsService, // Injetar metrics service (opcional)
  ) {
    this.logger.setContext('HTTP');
  }

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const route = url.split('?')[0]; // Remove query params
    const now = Date.now();

    this.logger.log(`→ ${method} ${url}`, 'HTTP');

    // Iniciar timer para métricas (se metrics service estiver disponível)
    let timer: (() => number) | null = null;
    if (this.metrics) {
      timer = this.metrics.httpRequestDuration.startTimer({
        method,
        route,
      });
    }

    return next.handle().pipe(
      tap({
        next: (data) => {
          const response = context.switchToHttp().getResponse();
          const { statusCode } = response;
          const time = Date.now() - now;

          // Registrar métricas (se disponível)
          if (this.metrics) {
            this.metrics.httpRequestCounter.inc({
              method,
              route,
              status: statusCode,
            });
            if (timer) timer(); // Finalizar timer
          }

          this.logger.log(
            `← ${method} ${url} ${statusCode} - ${time}ms`,
            'HTTP',
          );
        },
        error: (error) => {
          const time = Date.now() - now;
          const statusCode = error.status || 500;
          
          // Registrar métricas de erro (se disponível)
          if (this.metrics) {
            if (timer) timer(); // Finalizar timer mesmo em caso de erro
            this.metrics.httpRequestCounter.inc({
              method,
              route,
              status: statusCode,
            });
          }

          this.logger.error(
            `${method} ${url} ${statusCode} - ${time}ms - ${error.message}`,
            error.stack,
            'HTTP',
          );
        },
      }),
    );
  }
}
```

### 5.7 Atualizar App Module

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLoggerService } from './common/logger/my-logger.service';
import { LoggingInterceptor } from './common/interceptors/logging.interceptor';
import { HealthModule } from './health/health.module';
import { MetricsModule } from './metrics/metrics.module';

@Module({
  imports: [
    HealthModule,
    MetricsModule, // Adicionar metrics module
  ],
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: MyLoggerService,
      useValue: new MyLoggerService(),
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

### 5.8 Testar Métricas

```bash
# Fazer algumas requisições
curl http://localhost:3001/api/payments
curl http://localhost:3001/health

# Ver métricas
curl http://localhost:3001/metrics
```

**Exemplo de saída:**
```
# HELP http_requests_total Total de requisições HTTP
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/api/payments",status="200"} 1

# HELP http_request_duration_seconds Duração das requisições HTTP em segundos
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/api/payments",le="0.1"} 1
http_request_duration_seconds_bucket{method="GET",route="/api/payments",le="0.5"} 1
```

## Passo 6: Usar Logger nos Services

### 6.1 Exemplo: Usar Logger em um Service

Vamos criar um exemplo simples para demonstrar o uso do logger. Editar `src/app.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { MyLoggerService } from './common/logger/my-logger.service';

@Injectable()
export class AppService {
  constructor(private readonly logger: MyLoggerService) {
    // Define contexto do logger
    this.logger.setContext('AppService');
  }

  getHello(): string {
    this.logger.log('Método getHello() chamado', 'AppService');
    return 'Hello World!';
  }

  exemploComErro() {
    try {
      this.logger.log('Tentando executar operação...', 'AppService');
      // Simular erro
      throw new Error('Erro de exemplo');
    } catch (error) {
      this.logger.error(
        `Erro ao executar operação: ${error.message}`,
        error.stack,
      );
      throw error;
    }
  }
}
```

**Explicação:**

- `setContext()`: Define contexto do logger (aparece nos logs)
- `log()`: Registra informações normais
- `error()`: Registra erros com stack trace

## Passo 7: Remover console.log do main.ts

### 7.1 Atualizar main.ts

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MyLoggerService } from './common/logger/my-logger.service';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // Usar logger customizado
    logger: new MyLoggerService(),
  });

  // Habilitar CORS (opcional)
  app.enableCors();

  // Usar logger ao invés de console.log
  const logger = app.get(MyLoggerService);
  logger.setContext('Bootstrap');
  
  const port = process.env.PORT || 3000;
  
  logger.log(`API iniciada na porta ${port}`);
  logger.log(`Health check disponível em: http://localhost:${port}/health`);
  logger.log(`Métricas disponíveis em: http://localhost:${port}/metrics`);

  await app.listen(port);
}
bootstrap();
```

## Passo 8: Testar Observabilidade Completa

### 8.1 Iniciar Aplicação

```bash
npm run start:dev
```

### 8.2 Fazer Requisições

```bash
# Health check
curl http://localhost:3000/health

# Endpoint padrão
curl http://localhost:3000

# Ver métricas
curl http://localhost:3000/metrics
```

### 8.3 Verificar Logs

**Saída esperada no console:**
```
[HTTP] → GET /health
[HealthCheck] Health check executado
[HTTP] ← GET /health 200 - 12ms

[HTTP] → GET /
[AppService] Método getHello() chamado
[HTTP] ← GET / 200 - 5ms
```

### 8.4 Verificar Métricas

```bash
curl http://localhost:3000/metrics | grep http_requests
```

**Saída esperada:**
```
http_requests_total{method="GET",route="/health",status="200"} 1
http_requests_total{method="GET",route="/",status="200"} 1
```

## Conclusão

Este tutorial demonstrou como implementar **observabilidade profissional** em NestJS, seguindo boas práticas e padrões da indústria.

**Principais Benefícios:**

- **Visibilidade**: Entenda o que está acontecendo no sistema
- **Debugging**: Identifique problemas rapidamente
- **Performance**: Monitore e otimize latência
- **Confiabilidade**: Detecte problemas antes que afetem usuários

**Lembre-se:**

> **"Observabilidade não é opcional em sistemas distribuídos. É essencial para entender, diagnosticar e otimizar aplicações em produção."**


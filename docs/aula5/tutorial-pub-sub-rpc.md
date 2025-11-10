# Tutorial: RPC Pattern (Request/Response) com NestJS e RabbitMQ

## Introdução

Este tutorial demonstra como implementar o padrão **RPC (Remote Procedure Call)** usando NestJS e RabbitMQ. Este padrão permite comunicação síncrona via mensageria, onde:

- **Cliente** envia requisição e **espera** resposta
- **Servidor** processa requisição e retorna resposta
- Similar a HTTP, mas usando message broker

**Analogia:** É como fazer uma ligação telefônica. Você liga (envia requisição), aguarda a resposta, recebe a resposta e desliga. É uma comunicação bidirecional síncrona.

## Pré-requisitos

- Node.js 18 ou superior
- npm ou yarn
- Conhecimento básico de TypeScript e NestJS
- Conta no [CloudAMQP](https://www.cloudamqp.com/) (gratuita) ou RabbitMQ local
- Conhecimento do Event Pattern (recomendado ter feito o tutorial anterior)

## Passo 1: Estrutura do Projeto

Vamos criar 2 projetos NestJS separados:

```bash
# Criar diretório principal
mkdir nestjs-rpc-tutorial
cd nestjs-rpc-tutorial

# Criar os 2 projetos
nest new rpc-client
nest new rpc-server
```

**Estrutura final:**
```
nestjs-rpc-tutorial/
├── rpc-client/
└── rpc-server/
```

## Passo 2: Configurar RabbitMQ no CloudAMQP

### Criar Instância no CloudAMQP

1. Acesse [CloudAMQP](https://customer.cloudamqp.com/instance)
2. Faça login na sua conta (ou crie uma gratuita)
3. Crie uma nova instância (plano Little Lemur é gratuito)
4. Após criar, acesse a instância e copie a **URL de conexão**
   - Formato: `amqps://user:pass@host/vhost`
   - Exemplo: `amqps://wrsabynq:iunGpc5ecGcyA1eZPF71q6PE6LC8UleQ@jackal.rmq.cloudamqp.com/wrsabynq`

**Vantagens do CloudAMQP:**
- Não precisa instalar nada localmente
- Já vem configurado e pronto para usar
- Gratuito para testes (plano Little Lemur)
- Interface web para monitoramento
- Gerenciado pela CloudAMQP (sem preocupação com infraestrutura)

## Passo 3: Configurar RPC Client (Cliente)

### 3.1 Instalar Dependências

```bash
cd rpc-client
npm install @nestjs/microservices amqplib amqp-connection-manager
```

### 3.2 Criar Módulo de RPC Client

```bash
nest g module rpc-client
nest g service rpc-client --no-spec
nest g controller rpc-client --no-spec
```

### 3.3 Implementar RPC Client Service

Editar `src/rpc-client/rpc-client.service.ts`:

```typescript
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import {
  ClientProxy,
  ClientProxyFactory,
  Transport,
} from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class RpcClientService implements OnModuleInit {
  private readonly logger = new Logger(RpcClientService.name);
  private readonly client: ClientProxy;

  constructor() {
    // Configurar cliente RabbitMQ
    this.client = ClientProxyFactory.create({
      transport: Transport.RMQ,
      options: {
        urls: [
          // Coloque a url de conexão do rabbit aqui
          process.env.RABBITMQ_URL || 'coloque a url de conexão do rabbit aqui',
        ],
        queue: 'rpc_queue', // Fila para requisições RPC
        queueOptions: { durable: true },
      },
    });
  }

  async onModuleInit() {
    // Conectar ao RabbitMQ quando módulo iniciar
    await this.client.connect();
    this.logger.log('✅ RPC Client conectado ao RabbitMQ');
  }

  /**
   * RPC Pattern - Request/Response
   * Envia requisição e espera resposta
   * O NestJS cria automaticamente uma fila temporária para receber a resposta
   */
  async calcularSoma(data: { a: number; b: number }) {
    this.logger.log(`📤 Enviando RPC request: calcular soma de ${data.a} + ${data.b}`);
    
    try {
      // send() envia requisição e espera resposta
      const result = await firstValueFrom<{ result: number; timestamp: string }>(
        this.client.send('rpc_calcular', data),
      );
      
      this.logger.log(`📥 RPC response recebida:`, result);
      
      return {
        success: true,
        message: 'Cálculo realizado com sucesso',
        result: result.result,
        timestamp: result.timestamp,
      };
    } catch (error) {
      this.logger.error('❌ Erro no RPC call', error);
      throw error;
    }
  }

  /**
   * RPC para buscar informações de usuário
   */
  async buscarUsuario(userId: number) {
    this.logger.log(`📤 Enviando RPC request: buscar usuário ${userId}`);
    
    try {
      const result = await firstValueFrom<{ id: number; name: string; email: string }>(
        this.client.send('rpc_buscar_usuario', { userId }),
      );
      
      this.logger.log(`📥 RPC response recebida:`, result);
      
      return {
        success: true,
        user: result,
      };
    } catch (error) {
      this.logger.error('❌ Erro no RPC call', error);
      throw error;
    }
  }

  /**
   * RPC para processar pagamento
   */
  async processarPagamento(data: { pedidoId: number; valor: number; metodo: string }) {
    this.logger.log(`📤 Enviando RPC request: processar pagamento`, data);
    
    try {
      const result = await firstValueFrom<{ success: boolean; transactionId: string }>(
        this.client.send('rpc_processar_pagamento', data),
      );
      
      this.logger.log(`📥 RPC response recebida:`, result);
      
      return result;
    } catch (error) {
      this.logger.error('❌ Erro no RPC call', error);
      throw error;
    }
  }
}
```

**Explicação:**

- `client.send()`: Envia requisição RPC e **espera** resposta
- `firstValueFrom()`: Converte Observable do RxJS em Promise
- NestJS cria automaticamente uma fila temporária para receber a resposta
- Cliente bloqueia até receber resposta (comportamento síncrono)

### 3.4 Implementar RPC Client Controller

Editar `src/rpc-client/rpc-client.controller.ts`:

```typescript
import { Body, Controller, Post, Get, Param } from '@nestjs/common';
import { RpcClientService } from './rpc-client.service';

@Controller('rpc')
export class RpcClientController {
  constructor(private readonly rpcClientService: RpcClientService) {}

  /**
   * Endpoint para calcular soma via RPC
   * POST /rpc/calcular
   * Body: { "a": 10, "b": 5 }
   */
  @Post('calcular')
  async calcular(@Body() data: { a: number; b: number }) {
    return this.rpcClientService.calcularSoma(data);
  }

  /**
   * Endpoint para buscar usuário via RPC
   * GET /rpc/usuario/:id
   */
  @Get('usuario/:id')
  async buscarUsuario(@Param('id') id: string) {
    return this.rpcClientService.buscarUsuario(parseInt(id));
  }

  /**
   * Endpoint para processar pagamento via RPC
   * POST /rpc/pagamento
   * Body: { "pedidoId": 123, "valor": 99.90, "metodo": "cartao" }
   */
  @Post('pagamento')
  async processarPagamento(@Body() data: { pedidoId: number; valor: number; metodo: string }) {
    return this.rpcClientService.processarPagamento(data);
  }
}
```

### 3.5 Registrar Módulo

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { RpcClientModule } from './rpc-client/rpc-client.module';

@Module({
  imports: [RpcClientModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

### 3.6 Configurar Porta

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const port = process.env.PORT || 3000;
  await app.listen(port);
  
  console.log(`📤 RPC Client rodando na porta ${port}`);
  console.log(`📡 Endpoints disponíveis:`);
  console.log(`   - POST http://localhost:${port}/rpc/calcular`);
  console.log(`   - GET http://localhost:${port}/rpc/usuario/:id`);
  console.log(`   - POST http://localhost:${port}/rpc/pagamento`);
}
bootstrap();
```

## Passo 4: Configurar RPC Server (Servidor)

### 4.1 Instalar Dependências

```bash
cd ../rpc-server
npm install @nestjs/microservices amqplib amqp-connection-manager
```

### 4.2 Implementar RPC Server Controller

Editar `src/app.controller.ts`:

```typescript
import { Controller, Logger } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { AppService } from './app.service';

@Controller()
export class AppController {
  private readonly logger = new Logger(AppController.name);

  constructor(private readonly appService: AppService) {}

  /**
   * RPC Pattern - Request/Response
   * Escuta requisições RPC no padrão 'rpc_calcular'
   * Retorna resposta ao cliente
   * Apenas um servidor processa cada requisição
   */
  @MessagePattern('rpc_calcular')
  handleCalcular(@Payload() data: { a: number; b: number }) {
    this.logger.log(`📨 RPC request recebida: calcular ${data.a} + ${data.b}`);
    
    // Processar e retornar resposta
    const result = this.appService.calcularSoma(data.a, data.b);
    
    return {
      result: result,
      timestamp: new Date().toISOString(),
    };
  }

  /**
   * RPC para buscar usuário
   */
  @MessagePattern('rpc_buscar_usuario')
  handleBuscarUsuario(@Payload() data: { userId: number }) {
    this.logger.log(`📨 RPC request recebida: buscar usuário ${data.userId}`);
    
    const usuario = this.appService.buscarUsuario(data.userId);
    
    return usuario;
  }

  /**
   * RPC para processar pagamento
   */
  @MessagePattern('rpc_processar_pagamento')
  handleProcessarPagamento(@Payload() data: { pedidoId: number; valor: number; metodo: string }) {
    this.logger.log(`📨 RPC request recebida: processar pagamento`, data);
    
    const resultado = this.appService.processarPagamento(data);
    
    return resultado;
  }
}
```

**Explicação:**

- `@MessagePattern()`: Escuta requisições RPC (espera resposta)
- `@Payload()`: Extrai o payload da requisição
- **Retornar objeto** envia resposta ao cliente
- Apenas um servidor processa cada requisição (diferente do Event Pattern)

### 4.3 Implementar RPC Server Service

Editar `src/app.service.ts`:

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  // Simulação de banco de dados de usuários
  private readonly usuarios = [
    { id: 1, name: 'Ana Silva', email: 'ana@email.com' },
    { id: 2, name: 'Lucas Santos', email: 'lucas@email.com' },
    { id: 3, name: 'Maria Oliveira', email: 'maria@email.com' },
  ];

  /**
   * Calcula soma de dois números
   */
  calcularSoma(a: number, b: number): number {
    this.logger.log(`🔢 Calculando: ${a} + ${b}`);
    return a + b;
  }

  /**
   * Busca usuário por ID
   */
  buscarUsuario(userId: number): { id: number; name: string; email: string } | null {
    this.logger.log(`🔍 Buscando usuário ${userId}`);
    
    const usuario = this.usuarios.find(u => u.id === userId);
    
    if (!usuario) {
      this.logger.warn(`⚠️ Usuário ${userId} não encontrado`);
      return null;
    }
    
    return usuario;
  }

  /**
   * Processa pagamento
   */
  processarPagamento(data: { pedidoId: number; valor: number; metodo: string }): { success: boolean; transactionId: string } {
    this.logger.log(`💳 Processando pagamento:`, data);
    
    // Simular processamento de pagamento
    // Em produção, aqui você chamaria um gateway de pagamento real
    
    const transactionId = `TXN-${Date.now()}-${data.pedidoId}`;
    
    // Simular validação
    if (data.valor <= 0) {
      this.logger.error(`❌ Valor inválido: ${data.valor}`);
      return {
        success: false,
        transactionId: '',
      };
    }
    
    // Simular sucesso
    this.logger.log(`✅ Pagamento processado: ${transactionId}`);
    
    return {
      success: true,
      transactionId: transactionId,
    };
  }

  getHello(): string {
    return 'Hello World!';
  }
}
```

### 4.4 Configurar Conexão com RabbitMQ

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Conectar ao RabbitMQ para RECEBER requisições RPC
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      urls: [
        // Coloque a url de conexão do rabbit aqui
        process.env.RABBITMQ_URL || 'coloque a url de conexão do rabbit aqui',
      ],
      queue: 'rpc_queue', // Deve ser igual ao cliente
      queueOptions: { durable: true },
    },
  });

  // Iniciar microservice (escuta requisições RPC)
  await app.startAllMicroservices();
  
  // Iniciar aplicação HTTP (opcional, para health checks)
  const port = process.env.PORT || 3001;
  await app.listen(port);
  
  console.log(`📥 RPC Server rodando na porta ${port}`);
  console.log(`👂 Escutando requisições RPC na fila 'rpc_queue'`);
}
bootstrap();
```

## Passo 5: Testar o Sistema

### 5.1 Iniciar os Serviços

**Terminal 1 - RPC Server:**
```bash
cd rpc-server
npm run start:dev
```

**Saída esperada:**
```
📥 RPC Server rodando na porta 3001
👂 Escutando requisições RPC na fila 'rpc_queue'
```

**Terminal 2 - RPC Client:**
```bash
cd rpc-client
npm run start:dev
```

**Saída esperada:**
```
✅ RPC Client conectado ao RabbitMQ
📤 RPC Client rodando na porta 3000
📡 Endpoints disponíveis:
   - POST http://localhost:3000/rpc/calcular
   - GET http://localhost:3000/rpc/usuario/:id
   - POST http://localhost:3000/rpc/pagamento
```

### 5.2 Testar RPC de Cálculo

**Enviar requisição RPC:**
```bash
curl -X POST http://localhost:3000/rpc/calcular \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 5}'
```

**Resposta esperada:**
```json
{
  "success": true,
  "message": "Cálculo realizado com sucesso",
  "result": 15,
  "timestamp": "2024-01-01T10:00:00.000Z"
}
```

**Verificar no terminal do Client:**
```
📤 Enviando RPC request: calcular soma de 10 + 5
📥 RPC response recebida: { result: 15, timestamp: '...' }
```

**Verificar no terminal do Server:**
```
📨 RPC request recebida: calcular 10 + 5
🔢 Calculando: 10 + 5
```

**Observação importante:** O cliente **aguarda** a resposta antes de retornar!

### 5.3 Testar RPC de Busca de Usuário

**Buscar usuário:**
```bash
curl http://localhost:3000/rpc/usuario/1
```

**Resposta esperada:**
```json
{
  "success": true,
  "user": {
    "id": 1,
    "name": "Ana Silva",
    "email": "ana@email.com"
  }
}
```

**Buscar usuário inexistente:**
```bash
curl http://localhost:3000/rpc/usuario/999
```

**Resposta esperada:**
```json
{
  "success": true,
  "user": null
}
```

### 5.4 Testar RPC de Pagamento

**Processar pagamento:**
```bash
curl -X POST http://localhost:3000/rpc/pagamento \
  -H "Content-Type: application/json" \
  -d '{
    "pedidoId": 123,
    "valor": 99.90,
    "metodo": "cartao"
  }'
```

**Resposta esperada:**
```json
{
  "success": true,
  "transactionId": "TXN-1704110400000-123"
}
```

**Processar pagamento inválido:**
```bash
curl -X POST http://localhost:3000/rpc/pagamento \
  -H "Content-Type: application/json" \
  -d '{
    "pedidoId": 123,
    "valor": -10,
    "metodo": "cartao"
  }'
```

**Resposta esperada:**
```json
{
  "success": false,
  "transactionId": ""
}
```

### 5.5 Testar com Server Offline

**Cenário interessante:**

1. **Pare o RPC Server** (Ctrl+C)

2. **Envie requisição RPC:**
```bash
curl -X POST http://localhost:3000/rpc/calcular \
  -H "Content-Type: application/json" \
  -d '{"a": 10, "b": 5}'
```

3. **Cliente aguarda timeout** (pode demorar alguns segundos)

4. **Erro retornado:**
```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

**Diferença do Event Pattern:**
- No Event Pattern, mensagem é armazenada e processada depois
- No RPC Pattern, cliente **aguarda** resposta imediata
- Se servidor estiver offline, requisição falha

## Passo 6: Diferenças entre Event Pattern e RPC Pattern

### Comparação

| Característica | Event Pattern | RPC Pattern |
|----------------|---------------|-------------|
| **Espera resposta?** | ❌ Não (fire-and-forget) | ✅ Sim (aguarda) |
| **Múltiplos consumidores?** | ✅ Sim (Pub/Sub) | ❌ Não (1:1) |
| **Síncrono/Assíncrono?** | Assíncrono | Síncrono |
| **Quando usar?** | Eventos, notificações | Operações que precisam resultado |
| **Método** | `emit()` | `send()` |
| **Decorator** | `@EventPattern()` | `@MessagePattern()` |

### Quando Usar Cada Um

**Use Event Pattern quando:**
- ✅ Não precisa de resposta imediata
- ✅ Múltiplos serviços precisam processar o evento
- ✅ Processamento pode ser feito depois
- ✅ Quer desacoplamento total

**Exemplos:**
- Envio de emails
- Notificações push
- Atualização de cache
- Logs e métricas

**Use RPC Pattern quando:**
- ✅ Precisa de resposta imediata
- ✅ Operação requer resultado
- ✅ Substituição de chamadas HTTP síncronas
- ✅ Cálculos distribuídos

**Exemplos:**
- Buscar dados de usuário
- Processar pagamento e retornar ID
- Calcular valores
- Validar dados

## Passo 7: Adicionar Tratamento de Erros

### 7.1 Tratamento no Client

Editar `src/rpc-client/rpc-client.service.ts`:

```typescript
import { HttpException, HttpStatus } from '@nestjs/common';

async calcularSoma(data: { a: number; b: number }) {
  try {
    this.logger.log(`📤 Enviando RPC request: calcular soma de ${data.a} + ${data.b}`);
    
    const result = await firstValueFrom(
      this.client.send('rpc_calcular', data),
    );
    
    this.logger.log(`📥 RPC response recebida:`, result);
    
    return {
      success: true,
      message: 'Cálculo realizado com sucesso',
      result: result.result,
      timestamp: result.timestamp,
    };
  } catch (error) {
    this.logger.error('❌ Erro no RPC call', error);
    
    // Verificar se é timeout
    if (error.message?.includes('timeout')) {
      throw new HttpException(
        'Servidor RPC não respondeu a tempo. Tente novamente.',
        HttpStatus.GATEWAY_TIMEOUT,
      );
    }
    
    // Erro genérico
    throw new HttpException(
      'Erro ao processar requisição RPC. Tente novamente.',
      HttpStatus.INTERNAL_SERVER_ERROR,
    );
  }
}
```

### 7.2 Tratamento no Server

Editar `src/app.controller.ts`:

```typescript
@MessagePattern('rpc_calcular')
handleCalcular(@Payload() data: { a: number; b: number }) {
  try {
    this.logger.log(`📨 RPC request recebida: calcular ${data.a} + ${data.b}`);
    
    // Validar dados
    if (typeof data.a !== 'number' || typeof data.b !== 'number') {
      throw new Error('Parâmetros inválidos: a e b devem ser números');
    }
    
    const result = this.appService.calcularSoma(data.a, data.b);
    
    return {
      result: result,
      timestamp: new Date().toISOString(),
    };
  } catch (error) {
    this.logger.error('❌ Erro ao processar RPC', error);
    
    // Retornar erro para o cliente
    return {
      error: error.message,
      timestamp: new Date().toISOString(),
    };
  }
}
```

## Passo 8: Adicionar Validação com DTOs

### 8.1 Criar DTOs no Client

```bash
cd rpc-client
nest g class rpc-client/dto/calcular.dto
nest g class rpc-client/dto/pagamento.dto
```

**rpc-client/dto/calcular.dto.ts:**
```typescript
export class CalcularDto {
  a: number;
  b: number;
}
```

**rpc-client/dto/pagamento.dto.ts:**
```typescript
export class PagamentoDto {
  pedidoId: number;
  valor: number;
  metodo: string;
}
```

**Atualizar controller:**
```typescript
import { CalcularDto } from './dto/calcular.dto';
import { PagamentoDto } from './dto/pagamento.dto';

@Controller('rpc')
export class RpcClientController {
  @Post('calcular')
  async calcular(@Body() data: CalcularDto) {
    return this.rpcClientService.calcularSoma(data);
  }

  @Post('pagamento')
  async processarPagamento(@Body() data: PagamentoDto) {
    return this.rpcClientService.processarPagamento(data);
  }
}
```

## Resumo do Tutorial

### O que construímos:

1. **RPC Client** (porta 3000)
   - Endpoint: `POST /rpc/calcular` → calcula soma via RPC
   - Endpoint: `GET /rpc/usuario/:id` → busca usuário via RPC
   - Endpoint: `POST /rpc/pagamento` → processa pagamento via RPC

2. **RPC Server** (porta 3001)
   - Escuta requisições RPC na fila `rpc_queue`
   - Processa e retorna respostas
   - Apenas um servidor processa cada requisição

3. **RabbitMQ**
   - Gerencia filas e roteamento de requisições
   - Cria filas temporárias para respostas

### Conceitos Aprendidos:

✅ **RPC Pattern (Request/Response)**
- Cliente envia requisição e espera resposta
- Servidor processa e retorna resposta
- Síncrono (cliente bloqueia até receber resposta)
- Similar a HTTP, mas via mensageria

✅ **Comunicação Síncrona via Mensageria**
- Usa message broker mas mantém comportamento síncrono
- Útil quando precisa de resultado imediato
- Alternativa a chamadas HTTP diretas

✅ **Diferenças entre Event e RPC**
- Event: fire-and-forget, múltiplos consumidores
- RPC: request/response, um consumidor por requisição

### Quando Usar RPC Pattern:

- ✅ Operações que precisam de resposta imediata
- ✅ Buscar dados de outros serviços
- ✅ Processar e retornar resultado
- ✅ Substituição de chamadas HTTP síncronas
- ✅ Cálculos distribuídos

### Próximos Passos:

1. **Timeout Configurável**: Configurar timeout para requisições RPC
2. **Retry Logic**: Tentar novamente em caso de falha
3. **Circuit Breaker**: Proteger contra falhas em cascata
4. **Load Balancing**: Múltiplos servidores processando requisições
5. **Monitoring**: Métricas de latência e throughput

## Conclusão

Este tutorial demonstrou como implementar o **RPC Pattern (Request/Response)** usando NestJS e RabbitMQ. O sistema permite comunicação **síncrona** via mensageria, útil quando você precisa de resposta imediata.

**Principais Benefícios:**

- **Resposta Imediata**: Cliente recebe resultado na hora
- **Desacoplamento**: Usa message broker ao invés de HTTP direto
- **Confiabilidade**: Mensageria garante entrega
- **Flexibilidade**: Fácil adicionar novos servidores

**Analogia Final:**

> O RPC Pattern é como fazer uma ligação telefônica: você liga (envia requisição), aguarda a resposta, recebe a resposta e desliga. É uma comunicação bidirecional síncrona, onde você precisa do resultado antes de continuar.

**Diferença do Event Pattern:**

- **Event Pattern** = Rádio FM (transmite, não espera resposta, múltiplos ouvem)
- **RPC Pattern** = Telefone (liga, aguarda resposta, recebe e desliga)


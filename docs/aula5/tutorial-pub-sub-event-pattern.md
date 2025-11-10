# Tutorial: Event Pattern (Pub/Sub) com NestJS e RabbitMQ

## Introdução

Este tutorial demonstra como implementar o padrão **Event Pattern (Publish/Subscribe)** usando NestJS e RabbitMQ. Este é o padrão mais comum de mensageria, onde:

- **Publisher** publica eventos sem esperar resposta (fire-and-forget)
- **Subscribers** recebem e processam eventos assincronamente
- **Múltiplos subscribers** podem receber a mesma mensagem

**Analogia:** É como uma estação de rádio FM. A estação transmite (publisher), e todos os rádios sintonizados (subscribers) recebem a mesma música simultaneamente.

## Pré-requisitos

- Node.js 18 ou superior
- npm ou yarn
- Conhecimento básico de TypeScript e NestJS
- Conta no [CloudAMQP](https://www.cloudamqp.com/) (gratuita) ou RabbitMQ local

## Passo 1: Estrutura do Projeto

Vamos criar 2 projetos NestJS separados:

```bash
# Criar diretório principal
mkdir nestjs-event-pattern
cd nestjs-event-pattern

# Criar os 2 projetos
nest new publisher-service
nest new subscriber-service
```

**Estrutura final:**
```
nestjs-event-pattern/
├── publisher-service/
└── subscriber-service/
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

## Passo 3: Configurar Publisher Service

### 3.1 Instalar Dependências

```bash
cd publisher-service
npm install @nestjs/microservices amqplib amqp-connection-manager
```

**Dependências:**
- `@nestjs/microservices` → integração com message brokers
- `amqplib` → cliente RabbitMQ para Node.js
- `amqp-connection-manager` → gerenciamento de conexões

### 3.2 Criar Módulo de Publisher

```bash
nest g module publisher
nest g service publisher --no-spec
nest g controller publisher --no-spec
```

### 3.3 Implementar Publisher Service

Editar `src/publisher/publisher.service.ts`:

```typescript
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import {
  ClientProxy,
  ClientProxyFactory,
  Transport,
} from '@nestjs/microservices';

@Injectable()
export class PublisherService implements OnModuleInit {
  private readonly logger = new Logger(PublisherService.name);
  private readonly client: ClientProxy;

  constructor() {
    // Configurar cliente RabbitMQ
    this.client = ClientProxyFactory.create({
      transport: Transport.RMQ,
      options: {
        urls: [
          // Coloque a url de conexão do rabbit aqui
           'coloque a url de conexão do rabbit aqui',
        ],
        queue: 'eventos_pedidos',
        queueOptions: { durable: true }, // Fila persiste mesmo se broker reiniciar
      },
    });
  }

  async onModuleInit() {
    // Conectar ao RabbitMQ quando módulo iniciar
    await this.client.connect();
    this.logger.log('✅ Publisher conectado ao RabbitMQ');
  }

  /**
   * Event Pattern - Fire and Forget
   * Publica evento sem esperar resposta
   * Múltiplos subscribers podem receber a mesma mensagem
   */
  async publishEvent(eventType: string, data: any) {
    const event = {
      type: eventType,
      data: data,
      timestamp: new Date().toISOString(),
    };

    this.logger.log(`📤 Publicando evento: ${eventType}`, event);
    
    // emit() não espera resposta (fire-and-forget)
    await this.client.emit('eventos_pedidos', event);
    
    return {
      success: true,
      message: `Evento ${eventType} publicado com sucesso`,
      event: event,
    };
  }
}
```

**Explicação:**

- `ClientProxyFactory.create()`: Cria cliente para se comunicar com RabbitMQ
- `Transport.RMQ`: Especifica que usaremos RabbitMQ
- `queue: 'eventos_pedidos'`: Nome da fila onde eventos serão publicados
- `durable: true`: Fila persiste mesmo se RabbitMQ reiniciar
- `client.emit()`: Publica evento (fire-and-forget, não espera resposta)

### 3.4 Implementar Publisher Controller

Editar `src/publisher/publisher.controller.ts`:

```typescript
import { Body, Controller, Post } from '@nestjs/common';
import { PublisherService } from './publisher.service';

@Controller('events')
export class PublisherController {
  constructor(private readonly publisherService: PublisherService) {}

  /**
   * Endpoint para publicar evento de pedido criado
   * POST /events/pedido-criado
   * Body: { "pedidoId": 123, "usuarioId": 456, "valor": 99.90 }
   */
  @Post('pedido-criado')
  async publishPedidoCriado(@Body() data: { pedidoId: number; usuarioId: number; valor: number }) {
    return this.publisherService.publishEvent('pedido_criado', data);
  }

  /**
   * Endpoint para publicar evento de pedido cancelado
   * POST /events/pedido-cancelado
   * Body: { "pedidoId": 123, "motivo": "Cliente solicitou" }
   */
  @Post('pedido-cancelado')
  async publishPedidoCancelado(@Body() data: { pedidoId: number; motivo: string }) {
    return this.publisherService.publishEvent('pedido_cancelado', data);
  }

  /**
   * Endpoint genérico para publicar qualquer evento
   * POST /events
   * Body: { "eventType": "pedido_criado", "data": {...} }
   */
  @Post()
  async publishEvent(@Body() body: { eventType: string; data: any }) {
    return this.publisherService.publishEvent(body.eventType, body.data);
  }
}
```

### 3.5 Registrar Módulo

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PublisherModule } from './publisher/publisher.module';

@Module({
  imports: [PublisherModule],
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
  
  console.log(`📤 Publisher Service rodando na porta ${port}`);
  console.log(`📡 Endpoints disponíveis:`);
  console.log(`   - POST http://localhost:${port}/events/pedido-criado`);
  console.log(`   - POST http://localhost:${port}/events/pedido-cancelado`);
  console.log(`   - POST http://localhost:${port}/events`);
}
bootstrap();
```

## Passo 4: Configurar Subscriber Service

### 4.1 Instalar Dependências

```bash
cd ../subscriber-service
npm install @nestjs/microservices amqplib amqp-connection-manager
```

### 4.2 Implementar Subscriber Controller

Editar `src/app.controller.ts`:

```typescript
import { Controller, Logger } from '@nestjs/common';
import { EventPattern, Payload } from '@nestjs/microservices';
import { AppService } from './app.service';

@Controller()
export class AppController {
  private readonly logger = new Logger(AppController.name);

  constructor(private readonly appService: AppService) {}

  /**
   * Event Pattern - Fire and Forget
   * Escuta eventos publicados na fila 'eventos_pedidos'
   * Não retorna resposta ao publisher
   * Múltiplos subscribers podem receber a mesma mensagem
   */
  @EventPattern('eventos_pedidos')
  handleEvent(@Payload() event: { type: string; data: any; timestamp: string }) {
    this.logger.log(`📨 Evento recebido: ${event.type}`, event);

    // Processar evento baseado no tipo
    switch (event.type) {
      case 'pedido_criado':
        this.appService.processarPedidoCriado(event.data);
        break;
      case 'pedido_cancelado':
        this.appService.processarPedidoCancelado(event.data);
        break;
      default:
        this.logger.warn(`⚠️ Tipo de evento desconhecido: ${event.type}`);
    }
  }
}
```

**Explicação:**

- `@EventPattern()`: Escuta eventos na fila especificada
- `@Payload()`: Extrai o payload da mensagem
- Não retorna nada (fire-and-forget)
- Múltiplos subscribers podem receber a mesma mensagem

### 4.3 Implementar Subscriber Service

Editar `src/app.service.ts`:

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  /**
   * Processa evento de pedido criado
   */
  processarPedidoCriado(data: { pedidoId: number; usuarioId: number; valor: number }) {
    this.logger.log(`✅ Processando pedido criado:`, data);
    
    // Aqui você pode fazer qualquer processamento:
    // - Enviar email de confirmação
    // - Atualizar estoque
    // - Registrar métricas
    // - Notificar outros sistemas
    
    // Exemplo: simular envio de email
    this.logger.log(`📧 Email enviado para usuário ${data.usuarioId} sobre pedido ${data.pedidoId}`);
    
    // Exemplo: simular atualização de estoque
    this.logger.log(`📦 Estoque atualizado para pedido ${data.pedidoId}`);
    
    // Exemplo: simular registro de métrica
    this.logger.log(`📊 Métrica registrada: pedido criado no valor de R$ ${data.valor}`);
  }

  /**
   * Processa evento de pedido cancelado
   */
  processarPedidoCancelado(data: { pedidoId: number; motivo: string }) {
    this.logger.log(`✅ Processando pedido cancelado:`, data);
    
    // Exemplo: simular reembolso
    this.logger.log(`💰 Reembolso processado para pedido ${data.pedidoId}`);
    
    // Exemplo: simular liberação de estoque
    this.logger.log(`📦 Estoque liberado para pedido ${data.pedidoId}`);
    
    // Exemplo: simular envio de email
    this.logger.log(`📧 Email de cancelamento enviado para pedido ${data.pedidoId}`);
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

  // Conectar ao RabbitMQ para RECEBER eventos
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      urls: [
        // Coloque a url de conexão do rabbit aqui
         'coloque a url de conexão do rabbit aqui',
      ],
      queue: 'eventos_pedidos',
      queueOptions: { durable: true }, // Deve ser igual ao publisher-service
    },
  });

  // Iniciar microservice (escuta eventos)
  await app.startAllMicroservices();
  
  // Iniciar aplicação HTTP (opcional, para health checks)
  const port = process.env.PORT || 3001;
  await app.listen(port);
  
  console.log(`📥 Subscriber Service rodando na porta ${port}`);
  console.log(`👂 Escutando eventos na fila 'eventos_pedidos'`);
}
bootstrap();
```

**Explicação:**

- `app.connectMicroservice()`: Conecta aplicação ao RabbitMQ como microservice
- `app.startAllMicroservices()`: Inicia o listener de eventos
- A fila deve ter o mesmo nome e configuração (`durable: true`) em ambos os serviços

## Passo 5: Testar o Sistema

### 5.1 Iniciar os Serviços

**Terminal 1 - Subscriber Service:**
```bash
cd subscriber-service
npm run start:dev
```

**Saída esperada:**
```
📥 Subscriber Service rodando na porta 3001
👂 Escutando eventos na fila 'eventos_pedidos'
```

**Terminal 2 - Publisher Service:**
```bash
cd publisher-service
npm run start:dev
```

**Saída esperada:**
```
✅ Publisher conectado ao RabbitMQ
📤 Publisher Service rodando na porta 3000
📡 Endpoints disponíveis:
   - POST http://localhost:3000/events/pedido-criado
   - POST http://localhost:3000/events/pedido-cancelado
   - POST http://localhost:3000/events
```

### 5.2 Testar Publicação de Evento

**Publicar evento de pedido criado:**
```bash
curl -X POST http://localhost:3000/events/pedido-criado \
  -H "Content-Type: application/json" \
  -d '{
    "pedidoId": 123,
    "usuarioId": 456,
    "valor": 99.90
  }'
```

**Resposta do Publisher (imediata):**
```json
{
  "success": true,
  "message": "Evento pedido_criado publicado com sucesso",
  "event": {
    "type": "pedido_criado",
    "data": {
      "pedidoId": 123,
      "usuarioId": 456,
      "valor": 99.90
    },
    "timestamp": "2024-01-01T10:00:00.000Z"
  }
}
```

**Verificar no terminal do Subscriber:**
```
📨 Evento recebido: pedido_criado { type: 'pedido_criado', data: {...}, timestamp: '...' }
✅ Processando pedido criado: { pedidoId: 123, usuarioId: 456, valor: 99.90 }
📧 Email enviado para usuário 456 sobre pedido 123
📦 Estoque atualizado para pedido 123
📊 Métrica registrada: pedido criado no valor de R$ 99.90
```

**Observação importante:** O publisher retorna resposta **imediatamente**, sem esperar o processamento do subscriber!

### 5.3 Testar Múltiplos Eventos

**Publicar vários eventos rapidamente:**
```bash
# Evento 1
curl -X POST http://localhost:3000/events/pedido-criado \
  -H "Content-Type: application/json" \
  -d '{"pedidoId": 1, "usuarioId": 100, "valor": 50.00}'

# Evento 2
curl -X POST http://localhost:3000/events/pedido-criado \
  -H "Content-Type: application/json" \
  -d '{"pedidoId": 2, "usuarioId": 200, "valor": 150.00}'

# Evento 3 - Cancelamento
curl -X POST http://localhost:3000/events/pedido-cancelado \
  -H "Content-Type: application/json" \
  -d '{"pedidoId": 1, "motivo": "Cliente solicitou"}'
```

**Resultado:**
- Publisher retorna todas as respostas imediatamente
- Subscriber processa eventos assincronamente na ordem que chegam
- Não há bloqueio entre requisições

### 5.4 Testar com Subscriber Offline

**Cenário interessante:**

1. **Pare o Subscriber Service** (Ctrl+C)

2. **Publique eventos:**
```bash
curl -X POST http://localhost:3000/events/pedido-criado \
  -H "Content-Type: application/json" \
  -d '{"pedidoId": 999, "usuarioId": 888, "valor": 200.00}'
```

3. **Publisher retorna sucesso** (evento foi publicado no broker)

4. **Inicie o Subscriber novamente:**
```bash
cd subscriber-service
npm run start:dev
```

5. **Mensagem é processada automaticamente!** 🎉

**Por quê?**
- RabbitMQ armazena eventos em filas duráveis
- Quando subscriber volta online, eventos são entregues
- Garantia de entrega mesmo com falhas temporárias

## Passo 6: Múltiplos Subscribers (Pub/Sub Real)

### 6.1 Criar Segundo Subscriber

```bash
# Criar novo projeto
nest new subscriber-service-2
cd subscriber-service-2
npm install @nestjs/microservices amqplib amqp-connection-manager
```

**Configurar igual ao primeiro subscriber, mas mudar a porta:**

Editar `src/main.ts`:
```typescript
const port = process.env.PORT || 3002;
```

**Editar `src/app.service.ts` para diferenciar:**
```typescript
processarPedidoCriado(data: { pedidoId: number; usuarioId: number; valor: number }) {
  this.logger.log(`[SUBSCRIBER-2] ✅ Processando pedido criado:`, data);
  this.logger.log(`[SUBSCRIBER-2] 📊 Analytics: pedido ${data.pedidoId} criado`);
}
```

### 6.2 Testar Pub/Sub com Múltiplos Subscribers

**Iniciar todos os serviços:**
- Terminal 1: `subscriber-service` (porta 3001)
- Terminal 2: `subscriber-service-2` (porta 3002)
- Terminal 3: `publisher-service` (porta 3000)

**Publicar evento:**
```bash
curl -X POST http://localhost:3000/events/pedido-criado \
  -H "Content-Type: application/json" \
  -d '{"pedidoId": 555, "usuarioId": 777, "valor": 300.00}'
```

**Resultado:**
- **Ambos os subscribers recebem a mesma mensagem!**
- Cada um processa independentemente
- Isso é Pub/Sub real: **uma mensagem, múltiplos consumidores**

**Terminal Subscriber 1:**
```
📨 Evento recebido: pedido_criado
✅ Processando pedido criado: { pedidoId: 555, ... }
📧 Email enviado...
```

**Terminal Subscriber 2:**
```
📨 Evento recebido: pedido_criado
[SUBSCRIBER-2] ✅ Processando pedido criado: { pedidoId: 555, ... }
[SUBSCRIBER-2] 📊 Analytics: pedido 555 criado
```

## Passo 7: Adicionar Tratamento de Erros

### 7.1 Tratamento no Publisher

Editar `src/publisher/publisher.service.ts`:

```typescript
async publishEvent(eventType: string, data: any) {
  try {
    const event = {
      type: eventType,
      data: data,
      timestamp: new Date().toISOString(),
    };

    this.logger.log(`📤 Publicando evento: ${eventType}`, event);
    await this.client.emit('eventos_pedidos', event);
    
    return {
      success: true,
      message: `Evento ${eventType} publicado com sucesso`,
      event: event,
    };
  } catch (error) {
    this.logger.error('❌ Erro ao publicar evento', error);
    throw new HttpException(
      'Erro ao publicar evento. Tente novamente.',
      HttpStatus.INTERNAL_SERVER_ERROR,
    );
  }
}
```

### 7.2 Tratamento no Subscriber

Editar `src/app.controller.ts`:

```typescript
@EventPattern('eventos_pedidos')
handleEvent(@Payload() event: { type: string; data: any; timestamp: string }) {
  try {
    this.logger.log(`📨 Evento recebido: ${event.type}`, event);

    switch (event.type) {
      case 'pedido_criado':
        this.appService.processarPedidoCriado(event.data);
        break;
      case 'pedido_cancelado':
        this.appService.processarPedidoCancelado(event.data);
        break;
      default:
        this.logger.warn(`⚠️ Tipo de evento desconhecido: ${event.type}`);
    }
  } catch (error) {
    this.logger.error('❌ Erro ao processar evento', error);
    // Em produção, você pode:
    // - Enviar para Dead Letter Queue
    // - Notificar sistema de monitoramento
    // - Retry com backoff
  }
}
```

## Passo 8: Adicionar Validação com DTOs

### 8.1 Criar DTOs

```bash
cd publisher-service
nest g class publisher/dto/pedido-criado.dto
nest g class publisher/dto/pedido-cancelado.dto
```

**publisher/dto/pedido-criado.dto.ts:**
```typescript
export class PedidoCriadoDto {
  pedidoId: number;
  usuarioId: number;
  valor: number;
}
```

**publisher/dto/pedido-cancelado.dto.ts:**
```typescript
export class PedidoCanceladoDto {
  pedidoId: number;
  motivo: string;
}
```

**Atualizar controller:**
```typescript
import { PedidoCriadoDto } from './dto/pedido-criado.dto';
import { PedidoCanceladoDto } from './dto/pedido-cancelado.dto';

@Controller('events')
export class PublisherController {
  @Post('pedido-criado')
  async publishPedidoCriado(@Body() data: PedidoCriadoDto) {
    return this.publisherService.publishEvent('pedido_criado', data);
  }

  @Post('pedido-cancelado')
  async publishPedidoCancelado(@Body() data: PedidoCanceladoDto) {
    return this.publisherService.publishEvent('pedido_cancelado', data);
  }
}
```

## Resumo do Tutorial

### O que construímos:

1. **Publisher Service** (porta 3000)
   - Endpoint: `POST /events/pedido-criado` → publica evento
   - Endpoint: `POST /events/pedido-cancelado` → publica evento
   - Endpoint: `POST /events` → publica evento genérico

2. **Subscriber Service** (porta 3001)
   - Escuta eventos na fila `eventos_pedidos`
   - Processa eventos baseado no tipo
   - Múltiplos subscribers podem receber a mesma mensagem

3. **RabbitMQ**
   - Gerencia filas e roteamento de eventos
   - Garante entrega mesmo com falhas

### Conceitos Aprendidos:

✅ **Event Pattern (Fire-and-Forget)**
- Publisher não espera resposta
- Múltiplos subscribers podem receber
- Assíncrono e não-bloqueante
- Ideal para eventos e notificações

✅ **Comunicação Assíncrona**
- Serviços não precisam estar online simultaneamente
- Sistema continua funcionando com falhas parciais
- Melhor performance e escalabilidade

✅ **Pub/Sub Real**
- Uma mensagem pode ser recebida por múltiplos consumidores
- Cada subscriber processa independentemente
- Fácil adicionar novos subscribers

### Quando Usar Event Pattern:

- ✅ Envio de emails
- ✅ Notificações push
- ✅ Atualização de cache
- ✅ Logs e métricas
- ✅ Eventos de sistema
- ✅ Processamento em background

### Próximos Passos:

1. **Dead Letter Queue**: Fila para eventos que falharam
2. **Retry com Backoff**: Tentar novamente com delay crescente
3. **Idempotência**: Garantir que eventos duplicados não causem problemas
4. **Múltiplas Filas**: Diferentes tipos de eventos em filas diferentes
5. **Priorização**: Processar eventos urgentes primeiro
6. **Monitoring**: Métricas e observabilidade

## Conclusão

Este tutorial demonstrou como implementar o **Event Pattern (Pub/Sub)** usando NestJS e RabbitMQ. O sistema permite comunicação **assíncrona**, **desacoplada** e **confiável** entre serviços.

**Principais Benefícios:**

- **Desacoplamento**: Serviços não precisam conhecer detalhes uns dos outros
- **Resiliência**: Eventos são armazenados e entregues mesmo com falhas
- **Escalabilidade**: Fácil adicionar novos subscribers
- **Performance**: Comunicação assíncrona não bloqueia requisições
- **Flexibilidade**: Arquitetura orientada a eventos

**Analogia Final:**

> O Event Pattern é como uma estação de rádio FM: a estação transmite (publisher), e todos os rádios sintonizados (subscribers) recebem a mesma música simultaneamente. A estação não precisa saber quantos rádios estão ouvindo, e os rádios não precisam estar todos ligados ao mesmo tempo!


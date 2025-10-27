# Tutorial: Construindo uma Payment API com NestJS

## Introdução

Este tutorial demonstra como construir uma API de pagamentos completa usando NestJS, PostgreSQL e Prisma. A API será construída seguindo os princípios SOLID, com dois módulos: um que viola os princípios (legacy-payment) e outro que os aplica corretamente (payment).

## Pré-requisitos

- Node.js 18 ou superior
- Docker e Docker Compose
- Conhecimento básico de TypeScript e APIs REST

## Passo 1: Setup Inicial do Projeto

### 1.1 Criar Projeto NestJS

```bash
# Instalar Nest CLI globalmente
npm i -g @nestjs/cli

# Criar novo projeto
nest new payment-api
cd payment-api

# Instalar dependências adicionais
npm install @prisma/client prisma @nestjs/swagger swagger-ui-express
```

### 1.2 Configurar TypeScript

O NestJS já vem com TypeScript configurado, mas vamos verificar o `tsconfig.json`:

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "resolvePackageJsonExports": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2023",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

## Passo 2: Configuração do Banco de Dados

### 2.1 Configurar Banco de Dados

Você pode usar PostgreSQL de duas formas:

#### Opção 1: Docker Run (Mais Simples)

```bash
# Executar PostgreSQL com docker run
docker run --name payment-postgres \
  -e POSTGRES_USER=payment_user \
  -e POSTGRES_PASSWORD=payment_pass \
  -e POSTGRES_DB=payment_db \
  -p 5432:5432 \
  -d postgres:15-alpine

# Verificar se está rodando
docker ps

# Parar o container (quando necessário)
docker stop payment-postgres

# Iniciar novamente
docker start payment-postgres
```

#### Opção 2: Docker Compose (Mais Organizado)

Criar arquivo `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: payment-postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: payment_user
      POSTGRES_PASSWORD: payment_pass
      POSTGRES_DB: payment_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - payment-network

volumes:
  postgres_data:

networks:
  payment-network:
    driver: bridge
```

E executar:
```bash
# Subir banco
docker-compose up -d

# Parar banco
docker-compose down
```

**Recomendação**: Use a **Opção 1** (docker run) para simplicidade no tutorial.

### 2.2 Criar arquivo .env

**IMPORTANTE**: O arquivo `.env` deve ser criado na **raiz do projeto** (mesmo nível do `package.json`).

#### Opção 1: Via linha de comando

**Windows (PowerShell/CMD):**
```bash
echo "DATABASE_URL=\"postgresql://payment_user:payment_pass@localhost:5432/payment_db?schema=public\"" > .env
```

**Linux/Mac:**
```bash
echo 'DATABASE_URL="postgresql://payment_user:payment_pass@localhost:5432/payment_db?schema=public"' > .env
```

#### Opção 2: Via editor de texto (mais fácil)

1. **Crie um novo arquivo** na raiz do projeto
2. **Nomeie como** `.env` (com o ponto no início)
3. **Cole o conteúdo**:
```
DATABASE_URL="postgresql://payment_user:payment_pass@localhost:5432/payment_db?schema=public"
```

**Estrutura de pastas esperada:**
```
payment-api/
├── src/
├── package.json
├── .env          ← AQUI!
├── prisma/
└── ...
```

**Nota**: Certifique-se de que o PostgreSQL está rodando antes de continuar:
```bash
# Verificar se o container está rodando
docker ps

# Se não estiver, iniciar novamente
docker start payment-postgres
```

### 2.3 Configurar Schema do Prisma

Editar `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Payment {
  id            String   @id @default(uuid())
  type          String   // CREDIT_CARD, PIX, BOLETO
  amount        Decimal  @db.Decimal(10, 2)
  status        String   // PENDING, APPROVED, REJECTED
  customerName  String
  customerEmail String
  metadata      Json?    // Dados específicos de cada tipo
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  @@map("payments")
}
```

### 2.4 Aplicar Migrações

```bash
# Criar e aplicar migração
npx prisma migrate dev --name init

# Gerar cliente Prisma
npx prisma generate
```

## Passo 3: Criar Módulo Prisma

### 3.1 Criar PrismaService

```bash
# Criar módulo prisma
nest g module database
nest g service prisma --no-spec
```

Editar `src/database/prisma.service.ts`:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  // Conecta ao banco quando o módulo é inicializado
  async onModuleInit() {
    await this.$connect();
  }
}
```

Editar `src/database/database.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Module({
  providers: [PrismaService],
  exports: [PrismaService], // Exporta para outros módulos usarem
})
export class DatabaseModule {}
```

## Passo 4: Criar Módulo Legacy-Payment (Código Ruim - Sem SOLID)

Vamos começar criando o módulo que **viola** os princípios SOLID para depois comparar com a implementação correta.

### 4.1 Criar Módulo Legacy

```bash
# Criar módulo legacy-payment
nest g module legacy-payment
nest g service legacy-payment --no-spec
nest g controller legacy-payment --no-spec
```

### 4.2 Criar DTO Legacy

```bash
# Criar DTO usando NestJS CLI
nest g class legacy-payment/dto/create-legacy-payment.dto --no-spec
```

Editar `src/legacy-payment/dto/create-legacy-payment.dto.ts`:

```typescript
export class CreateLegacyPaymentDto {
  type: string;
  amount: number;
  customerName: string;
  customerEmail: string;
  cardNumber?: string;
  cvv?: string;
  expiryDate?: string;
  pixKey?: string;
  dueDate?: string;
}
```

### 4.3 Criar Service Legacy

Editar `src/legacy-payment/legacy-payment.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { CreateLegacyPaymentDto } from './dto/create-legacy-payment.dto/create-legacy-payment.dto';

@Injectable()
export class LegacyPaymentService {
  constructor(private prisma: PrismaService) {}

  // Método que viola todos os princípios SOLID
  async createPayment(createPaymentDto: CreateLegacyPaymentDto) {
    // Validação misturada com lógica de negócio
    if (!createPaymentDto.customerName) {
      throw new Error('Nome do cliente é obrigatório');
    }

    if (createPaymentDto.amount <= 0) {
      throw new Error('Valor deve ser maior que zero');
    }

    // if/else gigante - viola OCP
    let processedPayment;
    if (createPaymentDto.type === 'CREDIT_CARD') {
      processedPayment = this.processCreditCard(createPaymentDto);
    } else if (createPaymentDto.type === 'PIX') {
      processedPayment = this.processPix(createPaymentDto);
    } else if (createPaymentDto.type === 'BOLETO') {
      processedPayment = this.processBoleto(createPaymentDto);
    } else {
      throw new Error('Tipo de pagamento não suportado');
    }

    // Cálculo de taxa misturado
    const fee = this.calculateFee(
      createPaymentDto.amount,
      createPaymentDto.type,
    );
    const totalAmount = createPaymentDto.amount + fee;

    // Aplicação de regras misturada
    const status = this.determineStatus(
      createPaymentDto.amount,
      createPaymentDto.type,
    );

    // Formatação de dados misturada
    const formattedData = this.formatPaymentData(
      createPaymentDto,
      processedPayment,
      totalAmount,
      status,
    );

    // Persistência misturada
    const payment = await this.prisma.payment.create({
      data: {
        ...formattedData,
        // Converte metadata para o formato aceito pelo Prisma
        // eslint-disable-next-line @typescript-eslint/no-unsafe-assignment
        metadata: formattedData.metadata
          ? JSON.parse(JSON.stringify(formattedData.metadata))
          : undefined,
      },
    });

    // Envio de email separado do fluxo crítico, e aguardando completion
    this.sendEmail(payment);

    // Log misturado
    console.log(`Pagamento ${payment.id} criado com sucesso`);

    return payment;
  }

  // Métodos privados que fazem muitas coisas
  private processCreditCard(dto: CreateLegacyPaymentDto) {
    if (!dto.cardNumber || !dto.cvv || !dto.expiryDate) {
      throw new Error('Dados do cartão são obrigatórios');
    }

    return {
      cardNumber: dto.cardNumber.replace(/\d(?=\d{4})/g, '*'),
      cvv: '***',
      expiryDate: dto.expiryDate,
      processor: this.detectCardProcessor(dto.cardNumber),
      amount: dto.amount,
    };
  }

  private processPix(dto: CreateLegacyPaymentDto) {
    if (!dto.pixKey) {
      throw new Error('Chave PIX é obrigatória');
    }

    return {
      pixKey: dto.pixKey,
      qrCode: `pix-qr-code-${Date.now()}`,
      amount: dto.amount,
    };
  }

  private processBoleto(dto: CreateLegacyPaymentDto) {
    if (!dto.dueDate) {
      throw new Error('Data de vencimento é obrigatória');
    }

    return {
      boletoNumber: this.generateBoletoNumber(),
      dueDate: new Date(dto.dueDate),
      amount: dto.amount,
    };
  }

  private calculateFee(amount: number, type: string): number {
    if (type === 'CREDIT_CARD') {
      return amount * 0.03; // 3%
    } else if (type === 'PIX') {
      return 0; // PIX é gratuito
    } else if (type === 'BOLETO') {
      return 2.5; // Taxa fixa
    }
    return 0;
  }

  private determineStatus(amount: number, type: string): string {
    if (amount > 10000) {
      return 'PENDING';
    }
    if (type === 'CREDIT_CARD' && amount > 5000) {
      return 'PENDING';
    }
    return 'APPROVED';
  }

  private formatPaymentData(
    dto: CreateLegacyPaymentDto,
    processed: any,
    total: number,
    status: string,
  ) {
    return {
      type: dto.type,
      amount: total,
      status,
      customerName: dto.customerName.toUpperCase(),
      customerEmail: dto.customerEmail.toLowerCase(),
      metadata: processed as Record<string, unknown>,
    };
  }

  private detectCardProcessor(cardNumber: string): string {
    if (cardNumber.startsWith('4')) return 'Visa';
    if (cardNumber.startsWith('5')) return 'Mastercard';
    if (cardNumber.startsWith('3')) return 'American Express';
    return 'Unknown';
  }

  private generateBoletoNumber(): string {
    return '34191.79001 01043.510047 91020.150008 1 84460000020000';
  }

  private sendEmail(payment: any) {
    // eslint-disable-next-line @typescript-eslint/no-unsafe-member-access
    console.log(`📧 Email enviado para ${payment.customerEmail}`);
  }

  // Outros métodos que violam SRP
  async findAll() {
    const payments = await this.prisma.payment.findMany();
    return payments.map((p) => ({
      ...p,
      amount: `R$ ${Number(p.amount).toFixed(2)}`,
      createdAt: p.createdAt.toISOString(),
    }));
  }

  async findOne(id: string) {
    const payment = await this.prisma.payment.findUnique({ where: { id } });
    if (!payment) {
      throw new Error('Pagamento não encontrado');
    }
    return {
      ...payment,
      amount: `R$ ${Number(payment.amount).toFixed(2)}`,
      createdAt: payment.createdAt.toISOString(),
    };
  }

  async getStats() {
    const payments = await this.prisma.payment.findMany();
    const totalAmount = payments.reduce((sum, p) => sum + Number(p.amount), 0);
    const byType = payments.reduce((acc, p) => {
      // eslint-disable-next-line @typescript-eslint/no-unsafe-assignment
      acc[p.type] = (acc[p.type] || 0) + 1;
      return acc;
    }, {});

    return {
      totalPayments: payments.length,
      totalAmount: `R$ ${totalAmount.toFixed(2)}`,
      byType: Object.entries(byType).map(([type, count]) => ({
        type,
        count,
        percentage:
          (((count as number) / payments.length) * 100).toFixed(1) + '%',
      })),
    };
  }
}

```

### 4.4 Criar Controller Legacy

Editar `src/legacy-payment/legacy-payment.controller.ts`:

```typescript
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { ApiOperation, ApiResponse, ApiTags } from '@nestjs/swagger';
import { CreateLegacyPaymentDto } from './dto/create-legacy-payment.dto/create-legacy-payment.dto';
import { LegacyPaymentService } from './legacy-payment.service';

@ApiTags('legacy-payment')
@Controller('legacy-payment')
export class LegacyPaymentController {
  constructor(private legacyPaymentService: LegacyPaymentService) {}

  @Post()
  @ApiOperation({ summary: 'Processar pagamento (código ruim)' })
  @ApiResponse({ status: 201, description: 'Pagamento criado' })
  @ApiResponse({ status: 400, description: 'Dados inválidos' })
  async create(@Body() createPaymentDto: CreateLegacyPaymentDto) {
    return this.legacyPaymentService.createPayment(createPaymentDto);
  }

  @Get()
  @ApiOperation({ summary: 'Listar pagamentos (código ruim)' })
  @ApiResponse({ status: 200, description: 'Lista de pagamentos' })
  async findAll() {
    return this.legacyPaymentService.findAll();
  }

  @Get(':id')
  @ApiOperation({ summary: 'Buscar pagamento por ID (código ruim)' })
  @ApiResponse({ status: 200, description: 'Pagamento encontrado' })
  @ApiResponse({ status: 404, description: 'Pagamento não encontrado' })
  async findOne(@Param('id') id: string) {
    return this.legacyPaymentService.findOne(id);
  }

  @Get('stats/overview')
  @ApiOperation({ summary: 'Estatísticas (código ruim)' })
  @ApiResponse({ status: 200, description: 'Estatísticas calculadas' })
  async getStats() {
    return this.legacyPaymentService.getStats();
  }
}
```

### 4.5 Configurar Módulo Legacy

Editar `src/legacy-payment/legacy-payment.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from 'src/database/database.module';
import { LegacyPaymentController } from './legacy-payment.controller';
import { LegacyPaymentService } from './legacy-payment.service';

@Module({
  imports: [DatabaseModule],
  providers: [LegacyPaymentService],
  controllers: [LegacyPaymentController],
})
export class LegacyPaymentModule {}

```

## Passo 5: Criar Módulos de Estratégia (PIX, Cartão, Boleto)

### 5.1 Criar Módulo PIX

```bash
nest g module pix
nest g service pix --no-spec
```

Editar `src/pix/pix.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class PixService {
  // Processa pagamentos PIX
  async process(valor: number, metadados: Record<string, unknown>) {
    // Simula processamento de PIX
    const chavePix = metadados.chavePix as string;
    await new Promise((resolve) => setTimeout(resolve, 100));

    return {
      idTransacao: this.gerarIdTransacao(),
      status: 'PROCESSANDO',
      dadosAdicionais: {
        chavePix,
        qrCode: this.gerarQrCode(chavePix),
        valor,
      },
    };
  }

  // Calcula taxa do PIX (gratuita)
  calcularTaxa(): number {
    return 0;
  }

  // Retorna tipo de pagamento
  obterTipo(): string {
    return 'PIX';
  }

  // Gera QR Code para PIX
  private gerarQrCode(chavePix: string): string {
    return `pix-qr-code-${Date.now()}-${chavePix}`;
  }

  // Gera ID único para transação
  private gerarIdTransacao(): string {
    return `PIX_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

```

### 5.2 Criar Módulo PIX

Editar `src/pix/pix.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { PixService } from './pix.service';

@Module({
  providers: [PixService],
  exports: [PixService],
})
export class PixModule {}
```

### 5.3 Criar Módulo Cartão de Crédito

```bash
nest g module credit-card
nest g service credit-card --no-spec
```

Editar `src/credit-card/credit-card.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class CreditCardService {
  // Processa pagamentos com cartão
  async process(valor: number, metadados: Record<string, unknown>) {
    const numeroCartao = metadados.numeroCartao as string;
    await new Promise((resolve) => setTimeout(resolve, 100));
    
    return {
      idTransacao: this.gerarIdTransacao(),
      status: 'PROCESSANDO',
      dadosAdicionais: {
        numeroCartao: this.mascararNumeroCartao(numeroCartao),
        cvv: '***',
        dataVencimento: metadados.dataVencimento,
        processador: this.detectarProcessador(numeroCartao),
        valor,
      },
    };
  }

  // Calcula taxa do cartão (3%)
  calcularTaxa(valor: number): number {
    return valor * 0.03;
  }

  // Retorna tipo de pagamento
  obterTipo(): string {
    return 'CARTAO_CREDITO';
  }

  // Mascara número do cartão para segurança
  private mascararNumeroCartao(numeroCartao: string): string {
    return numeroCartao.replace(/\d(?=\d{4})/g, '*');
  }

  // Detecta processador do cartão
  private detectarProcessador(numeroCartao: string): string {
    if (numeroCartao.startsWith('4')) return 'Visa';
    if (numeroCartao.startsWith('5')) return 'Mastercard';
    if (numeroCartao.startsWith('3')) return 'American Express';
    return 'Desconhecido';
  }

  // Gera ID único para transação
  private gerarIdTransacao(): string {
    return `CC_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 5.4 Criar Módulo Cartão de Crédito

Editar `src/credit-card/credit-card.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { CreditCardService } from './credit-card.service';

@Module({
  providers: [CreditCardService],
  exports: [CreditCardService],
})
export class CreditCardModule {}
```

### 5.5 Criar Módulo Boleto

```bash
nest g module boleto
nest g service boleto --no-spec
```

Editar `src/boleto/boleto.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class BoletoService {
  // Processa pagamentos via boleto
  async process(valor: number, metadados: Record<string, unknown>) {
    const dataVencimento = metadados.dataVencimento as string;
    await new Promise((resolve) => setTimeout(resolve, 100));
    
    return {
      idTransacao: this.gerarIdTransacao(),
      status: 'PROCESSANDO',
      dadosAdicionais: {
        numeroBoleto: this.gerarNumeroBoleto(),
        dataVencimento: new Date(dataVencimento),
        valor,
      },
    };
  }

  // Calcula taxa do boleto (fixa)
  calcularTaxa(): number {
    return 2.5;
  }

  // Retorna tipo de pagamento
  obterTipo(): string {
    return 'BOLETO';
  }

  // Gera número do boleto
  private gerarNumeroBoleto(): string {
    return '34191.79001 01043.510047 91020.150008 1 84460000020000';
  }

  // Gera ID único para transação
  private gerarIdTransacao(): string {
    return `BOL_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 5.6 Criar Módulo Boleto

Editar `src/boleto/boleto.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { BoletoService } from './boleto.service';

@Module({
  providers: [BoletoService],
  exports: [BoletoService],
})
export class BoletoModule {}
```

## Passo 6: Criar Módulo Payment (Código Bom com SOLID)

### 6.1 Criar Interfaces

```bash
# Criar interfaces usando NestJS CLI
nest g interface payment/interfaces/payment-strategy.interface --no-spec
```

Editar `src/payment/interfaces/payment-strategy.interface/payment-strategy.interface.interface.ts`:

```typescript
// Interface para dados de processamento
export interface DadosProcessamentoPagamento {
  idTransacao: string;
  status: string;
  dadosAdicionais: Record<string, unknown>;
}

// Interface para dados de pagamento
export interface DadosPagamento {
  id: string;
  tipo: string;
  valor: number;
  status: string;
  nomeCliente: string;
  emailCliente: string;
  metadados: Record<string, unknown>;
  dataCriacao: Date;
  dataAtualizacao: Date;
}

// Interface para estatísticas
export interface EstatisticasPagamento {
  totalPagamentos: number;
  valorTotal: number;
  pagamentosAprovados: number;
  pagamentosPendentes: number;
  pagamentosRejeitados: number;
}

// Interface para estratégias de pagamento
export interface IPaymentStrategy {
  process(
    valor: number,
    metadados: Record<string, unknown>,
  ): Promise<DadosProcessamentoPagamento>;
  calcularTaxa(valor?: number): number;
  obterTipo(): string;
}

// Interface para repositório
export interface IPaymentRepository {
  criar(
    dados: Omit<DadosPagamento, 'id' | 'dataCriacao' | 'dataAtualizacao'>,
  ): Promise<DadosPagamento>;
  buscarPorId(id: string): Promise<DadosPagamento | null>;
  buscarTodos(): Promise<DadosPagamento[]>;
  obterEstatisticas(): Promise<EstatisticasPagamento>;
}

```

### 6.2 Criar DTOs

```bash
# Criar DTOs usando NestJS CLI
nest g class payment/dto/create-payment.dto --no-spec
```

Editar `src/payment/dto/create-payment.dto.ts`:

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreatePaymentDto {
  @ApiProperty({
    description: 'Tipo do pagamento',
    enum: ['CARTAO_CREDITO', 'PIX', 'BOLETO'],
    example: 'PIX',
  })
  tipo: string;

  @ApiProperty({
    description: 'Valor do pagamento em reais',
    example: 100.5,
    minimum: 0.01,
  })
  valor: number;

  @ApiProperty({
    description: 'Nome completo do cliente',
    example: 'João Silva',
  })
  nomeCliente: string;

  @ApiProperty({
    description: 'Email do cliente',
    example: 'joao@email.com',
    format: 'email',
  })
  emailCliente: string;

  @ApiProperty({
    description: 'Metadados específicos do tipo de pagamento',
    example: { chavePix: 'joao@email.com' },
    required: false,
  })
  metadados?: Record<string, unknown>;
}

```

### 6.3 Criar Repositório

```bash
# Criar repositório usando NestJS CLI
nest g class payment/repository/payment.repository --no-spec
```

Editar `src/payment/repository/payment.repository.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../../database/prisma.service';
import type {
  DadosPagamento,
  EstatisticasPagamento,
  IPaymentRepository,
} from '../../interfaces/payment-strategy.interface/payment-strategy.interface.interface';

@Injectable()
export class PaymentRepository implements IPaymentRepository {
  constructor(private prisma: PrismaService) {}

  // Cria novo pagamento no banco
  async criar(
    dados: Omit<DadosPagamento, 'id' | 'dataCriacao' | 'dataAtualizacao'>,
  ): Promise<DadosPagamento> {
    const pagamento = await this.prisma.payment.create({
      data: {
        type: dados.tipo,
        amount: dados.valor,
        status: dados.status,
        customerName: dados.nomeCliente,
        customerEmail: dados.emailCliente,
      },
    });

    return {
      id: pagamento.id,
      tipo: pagamento.type,
      valor: Number(pagamento.amount),
      status: pagamento.status,
      nomeCliente: pagamento.customerName,
      emailCliente: pagamento.customerEmail,
      metadados: pagamento.metadata as Record<string, unknown>,
      dataCriacao: pagamento.createdAt,
      dataAtualizacao: pagamento.updatedAt,
    };
  }

  // Busca pagamento por ID
  async buscarPorId(id: string): Promise<DadosPagamento | null> {
    const pagamento = await this.prisma.payment.findUnique({
      where: { id },
    });

    if (!pagamento) return null;

    return {
      id: pagamento.id,
      tipo: pagamento.type,
      valor: Number(pagamento.amount),
      status: pagamento.status,
      nomeCliente: pagamento.customerName,
      emailCliente: pagamento.customerEmail,
      metadados: pagamento.metadata as Record<string, unknown>,
      dataCriacao: pagamento.createdAt,
      dataAtualizacao: pagamento.updatedAt,
    };
  }

  // Busca todos os pagamentos
  async buscarTodos(): Promise<DadosPagamento[]> {
    const pagamentos = await this.prisma.payment.findMany({
      orderBy: { createdAt: 'desc' },
    });

    return pagamentos.map((pagamento) => ({
      id: pagamento.id,
      tipo: pagamento.type,
      valor: Number(pagamento.amount),
      status: pagamento.status,
      nomeCliente: pagamento.customerName,
      emailCliente: pagamento.customerEmail,
      metadados: pagamento.metadata as Record<string, unknown>,
      dataCriacao: pagamento.createdAt,
      dataAtualizacao: pagamento.updatedAt,
    }));
  }

  // Calcula estatísticas dos pagamentos
  async obterEstatisticas(): Promise<EstatisticasPagamento> {
    const pagamentos = await this.prisma.payment.findMany();

    const totalPagamentos = pagamentos.length;
    const valorTotal = pagamentos.reduce((sum, p) => sum + Number(p.amount), 0);
    const pagamentosAprovados = pagamentos.filter(
      (p) => p.status === 'APROVADO',
    ).length;
    const pagamentosPendentes = pagamentos.filter(
      (p) => p.status === 'PENDENTE',
    ).length;
    const pagamentosRejeitados = pagamentos.filter(
      (p) => p.status === 'REJEITADO',
    ).length;

    return {
      totalPagamentos,
      valorTotal,
      pagamentosAprovados,
      pagamentosPendentes,
      pagamentosRejeitados,
    };
  }
}


```

### 6.4 Criar Service Principal

```bash
nest g module payment
nest g service payment --no-spec
nest g controller payment --no-spec
```

Editar `src/payment/payment.service.ts`:

```typescript
import { Inject, Injectable } from '@nestjs/common';
import { BoletoService } from '../boleto/boleto.service';
import { CreditCardService } from '../credit-card/credit-card.service';
import { PixService } from '../pix/pix.service';
import { CreatePaymentDto } from './dto/create-payment.dto/create-payment.dto';
import type {
  DadosPagamento,
  EstatisticasPagamento,
  IPaymentRepository,
  IPaymentStrategy,
} from './interfaces/payment-strategy.interface/payment-strategy.interface.interface';

@Injectable()
export class PaymentService {
  private estrategias: Map<string, IPaymentStrategy>;

  constructor(
    @Inject('IPaymentRepository')
    private repositorioPagamento: IPaymentRepository,
    private servicoCartaoCredito: CreditCardService,
    private servicoPix: PixService,
    private servicoBoleto: BoletoService,
  ) {
    // Configura estratégias de pagamento
    this.estrategias = new Map();
    this.estrategias.set('CARTAO_CREDITO', servicoCartaoCredito);
    this.estrategias.set('PIX', servicoPix);
    this.estrategias.set('BOLETO', servicoBoleto);
  }

  // Cria novo pagamento
  async criarPagamento(
    dadosPagamento: CreatePaymentDto,
  ): Promise<DadosPagamento> {
    // Busca estratégia de pagamento
    const estrategia = this.estrategias.get(dadosPagamento.tipo);
    if (!estrategia) {
      throw new Error('Tipo de pagamento não suportado');
    }

    // Processa pagamento usando estratégia específica
    const pagamentoProcessado = await estrategia.process(
      dadosPagamento.valor,
      dadosPagamento.metadados || {},
    );

    // Calcula taxa usando estratégia
    const taxa = estrategia.calcularTaxa(dadosPagamento.valor);
    const valorTotal = dadosPagamento.valor + taxa;

    // Determina status do pagamento
    const status = this.determinarStatus(
      dadosPagamento.valor,
      dadosPagamento.tipo,
    );

    // Salva no banco de dados
    const pagamento = await this.repositorioPagamento.criar({
      tipo: dadosPagamento.tipo,
      valor: valorTotal,
      status,
      nomeCliente: dadosPagamento.nomeCliente,
      emailCliente: dadosPagamento.emailCliente,
      metadados: pagamentoProcessado.dadosAdicionais,
    });

    return pagamento;
  }

  // Busca todos os pagamentos
  async buscarTodos(): Promise<DadosPagamento[]> {
    return this.repositorioPagamento.buscarTodos();
  }

  // Busca pagamento por ID
  async buscarPorId(id: string): Promise<DadosPagamento | null> {
    return this.repositorioPagamento.buscarPorId(id);
  }

  // Obtém estatísticas
  async obterEstatisticas(): Promise<EstatisticasPagamento> {
    return this.repositorioPagamento.obterEstatisticas();
  }

  // Determina status baseado em regras de negócio
  private determinarStatus(valor: number, tipo: string): string {
    if (valor > 10000) {
      return 'PENDENTE'; // Precisa aprovação
    }
    if (tipo === 'CARTAO_CREDITO' && valor > 5000) {
      return 'PENDENTE'; // Cartão alto valor
    }
    return 'APROVADO';
  }
}
```

### 6.5 Criar Controller

Editar `src/payment/payment.controller.ts`:

```typescript
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { ApiOperation, ApiResponse, ApiTags } from '@nestjs/swagger';
import { CreatePaymentDto } from './dto/create-payment.dto/create-payment.dto';
import { PaymentService } from './payment.service';

@ApiTags('payment')
@Controller('pagamentos')
export class PaymentController {
  constructor(private servicoPagamento: PaymentService) {}

  @Post()
  @ApiOperation({ summary: 'Processar pagamento' })
  @ApiResponse({ status: 201, description: 'Pagamento criado com sucesso' })
  @ApiResponse({ status: 400, description: 'Dados inválidos' })
  async criar(@Body() dadosPagamento: CreatePaymentDto) {
    return await this.servicoPagamento.criarPagamento(dadosPagamento);
  }

  @Get()
  @ApiOperation({ summary: 'Listar todos os pagamentos' })
  @ApiResponse({ status: 200, description: 'Lista de pagamentos' })
  async buscarTodos() {
    return await this.servicoPagamento.buscarTodos();
  }

  @Get(':id')
  @ApiOperation({ summary: 'Buscar pagamento por ID' })
  @ApiResponse({ status: 200, description: 'Pagamento encontrado' })
  @ApiResponse({ status: 404, description: 'Pagamento não encontrado' })
  async buscarPorId(@Param('id') id: string) {
    return await this.servicoPagamento.buscarPorId(id);
  }

  @Get('estatisticas/visao-geral')
  @ApiOperation({ summary: 'Obter estatísticas de pagamentos' })
  @ApiResponse({ status: 200, description: 'Estatísticas calculadas' })
  async obterEstatisticas() {
    return await this.servicoPagamento.obterEstatisticas();
  }
}

```

### 6.6 Configurar Módulo Payment

Editar `src/payment/payment.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { BoletoModule } from '../boleto/boleto.module';
import { CreditCardModule } from '../credit-card/credit-card.module';
import { DatabaseModule } from '../database/database.module';
import { PixModule } from '../pix/pix.module';
import { PaymentController } from './payment.controller';
import { PaymentService } from './payment.service';
import { PaymentRepository } from './repository/payment.repository/payment.repository';

@Module({
  imports: [DatabaseModule, PixModule, CreditCardModule, BoletoModule],
  controllers: [PaymentController],
  providers: [
    PaymentService,
    {
      provide: 'IPaymentRepository',
      useClass: PaymentRepository,
    },
  ],
})
export class PaymentModule {}

```

## Passo 7: Configurar Swagger

### 7.1 Configurar main.ts

Editar `src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Configuração do Swagger
  const config = new DocumentBuilder()
    .setTitle('Payment API - SOLID')
    .setDescription('Demonstração dos princípios SOLID com NestJS')
    .setVersion('1.0')
    .addTag('legacy-payment', 'Módulo que viola SOLID (código ruim)')
    .addTag('payment', 'Módulo que aplica SOLID (código bom)')
    .build();

  const document = SwaggerModule.createDocument(app as any, config);
  SwaggerModule.setup('api', app as any, document);

  // eslint-disable-next-line @typescript-eslint/no-unsafe-call, @typescript-eslint/no-unsafe-member-access
  await (app as any).listen(3000);
}

bootstrap();

```

### 7.2 Atualizar AppModule

Editar `src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { PaymentModule } from './payment/payment.module';
import { LegacyPaymentModule } from './legacy-payment/legacy-payment.module';

@Module({
  imports: [DatabaseModule, PaymentModule, LegacyPaymentModule],
})
export class AppModule {}
```

## Passo 8: Testando a API

### 8.1 Iniciar Servidor

```bash
# Iniciar servidor de desenvolvimento
npm run start:dev
```

### 8.2 Verificação Rápida

Antes de testar os endpoints, confirme que tudo está funcionando:

```bash
# 1. Verificar se o banco está rodando
docker ps

# 2. Verificar se as dependências estão instaladas
npm list --depth=0

# 3. Verificar se o Prisma está configurado
npx prisma db push

# 4. Iniciar a aplicação
npm run start:dev
```

**Se tudo estiver OK, você verá:**

- Container PostgreSQL rodando
- Dependências instaladas sem erros
- Banco sincronizado com sucesso
- Servidor rodando em `http://localhost:3000`

### 8.3 Testar Endpoints via Swagger

Acesse a documentação interativa do Swagger em: `http://localhost:3000/api`

**Endpoints disponíveis:**

**Módulo Payment (código bom)** 

`/pagamentos`

- `POST /pagamentos`: Criar pagamento
- `GET /pagamentos`: Listar pagamentos
- `GET /pagamentos/estatisticas/visao-geral`: Ver estatísticas

**Módulo Legacy-Payment (código ruim)**

`/legacy-payment`

- `POST /legacy-payment`: Criar pagamento legacy
- `GET /legacy-payment`: Listar pagamentos legacy
- `GET /legacy-payment/stats/overview`: Ver estatísticas legacy

**Como testar:**

1. Abra o Swagger UI no navegador
2. Expanda o endpoint desejado
3. Clique em "Try it out"
4. Preencha os dados de exemplo
5. Clique em "Execute" para testar

## Estrutura Final do Projeto

Após seguir todo o tutorial, sua estrutura de projeto deve ficar assim:

```
payment-api/
├── src/
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   ├── main.ts
│   ├── database/
│   │   ├── database.module.ts
│   │   └── prisma.service.ts
│   ├── payment/
│   │   ├── dto/
│   │   │   └── create-payment.dto/
│   │   │       └── create-payment.dto.ts
│   │   ├── interfaces/
│   │   │   └── payment-strategy.interface/
│   │   │       └── payment-strategy.interface.interface.ts
│   │   ├── repository/
│   │   │   └── payment.repository/
│   │   │       └── payment.repository.ts
│   │   ├── payment.controller.ts
│   │   ├── payment.module.ts
│   │   └── payment.service.ts
│   ├── legacy-payment/
│   │   ├── dto/
│   │   │   └── create-legacy-payment.dto/
│   │   │       └── create-legacy-payment.dto.ts
│   │   ├── legacy-payment.controller.ts
│   │   ├── legacy-payment.module.ts
│   │   └── legacy-payment.service.ts
│   ├── pix/
│   │   ├── pix.module.ts
│   │   └── pix.service.ts
│   ├── credit-card/
│   │   ├── credit-card.module.ts
│   │   └── credit-card.service.ts
│   └── boleto/
│       ├── boleto.module.ts
│       └── boleto.service.ts
├── prisma/
│   ├── migrations/
│   │   ├── 20251027162427_init/
│   │   │   └── migration.sql
│   │   └── migration_lock.toml
│   └── schema.prisma
├── test/
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── .gitignore
├── eslint.config.mjs
├── nest-cli.json
├── package.json
├── package-lock.json
├── README.md
├── tsconfig.build.json
└── tsconfig.json
```

### Arquivos Principais

**Configuração:**

- `package.json` - Dependências do projeto
- `nest-cli.json` - Configuração do NestJS CLI
- `tsconfig.json` - Configuração do TypeScript
- `eslint.config.mjs` - Configuração do ESLint

**Banco de Dados:**

- `prisma/schema.prisma` - Schema do banco
- `prisma/migrations/20251027162427_init/` - Migração inicial
- `prisma/migration_lock.toml` - Lock de migrações

**Código Principal:**

- `src/main.ts` - Ponto de entrada da aplicação
- `src/app.module.ts` - Módulo principal
- `src/app.controller.ts` - Controller principal
- `src/app.controller.spec.ts` - Testes do controller
- `src/database/` - Configuração do Prisma

**Módulos de Pagamento:**

- `src/payment/` - Módulo SOLID (código bom)
  - `dto/create-payment.dto/` - DTOs de criação
  - `interfaces/payment-strategy.interface/` - Interface de estratégia
  - `repository/payment.repository/` - Repositório de pagamentos
- `src/legacy-payment/` - Módulo legacy (código ruim)
  - `dto/create-legacy-payment.dto/` - DTOs legacy
- `src/pix/`, `src/credit-card/`, `src/boleto/` - Serviços específicos

**Testes:**

- `test/app.e2e-spec.ts` - Testes end-to-end
- `test/jest-e2e.json` - Configuração do Jest para E2E



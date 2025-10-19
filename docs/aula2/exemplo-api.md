# Aula 2 — Exemplo Prático de API com NestJS

## Objetivo da Aula

Nesta aula vamos **colocar em prática** os conceitos vistos sobre APIs na aula anterior.  
A ideia é construir uma **API simples**, usando **NestJS** e **Prisma**, conectada a um banco **PostgreSQL**.  
Essa API permitirá **criar, listar, buscar, atualizar e deletar usuários** — um CRUD básico.

## Stack Utilizada

- **NestJS** → Framework Node.js para back-end modular e escalável.  
- **Prisma** → ORM moderno, facilita o acesso ao banco de dados.  
- **PostgreSQL** → Banco relacional usado na aplicação.  
- **Docker** *(opcional)* → Para subir o banco rapidamente.

## Estrutura da Aplicação

```
minha-api/
├─ prisma/
│  └─ schema.prisma
├─ src/
│  ├─ app.module.ts
│  ├─ main.ts
│  └─ users/
│     ├─ dto/
│     │  └─ create-user.dto.ts
│     ├─ users.controller.ts
│     ├─ users.service.ts
│     └─ users.module.ts
├─ .env
├─ package.json
├─ Dockerfile
└─ docker-compose.yml
```

## Passo 1 — Instalar PostgreSQL

### Opção 1: Docker (Recomendado)

```bash
# Baixar e executar PostgreSQL na porta 5433
docker run --name postgres-api -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=minha_api -p 5433:5432 -d postgres:15

# Verificar se está rodando
docker ps
```

### Opção 2: Instalação Local

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Criar usuário e banco
sudo -u postgres psql
CREATE USER postgres WITH PASSWORD 'postgres';
CREATE DATABASE minha_api OWNER postgres;
GRANT ALL PRIVILEGES ON DATABASE minha_api TO postgres;
\q

# Configurar para rodar na porta 5433
sudo nano /etc/postgresql/*/main/postgresql.conf
# Alterar: port = 5433
sudo systemctl restart postgresql
```

**macOS (Homebrew):**
```bash
brew install postgresql
brew services start postgresql
createdb minha_api
```

**Windows:**
```bash
# Baixar do site oficial: https://www.postgresql.org/download/windows/
# Durante instalação, configurar porta 5433
```

## Passo 2 — Criar o Projeto

```bash
npm i -g @nestjs/cli
nest new minha-api
cd minha-api

npm install @prisma/client
npm install -D prisma
npm install pg
npm install @nestjs/swagger swagger-ui-express
```

Crie um arquivo `.env` com o conteúdo:

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/minha_api?schema=public"
```


## Passo 3 — Criar o Modelo Prisma

`prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
}
```

Rode os comandos:

```bash
npx prisma migrate dev --name init
npx prisma generate
```

## Passo 4 — Criar o PrismaService

### Gerar arquivos:

```bash
nest generate module prisma
nest generate service prisma
```

### Arquivos gerados:

`src/prisma/prisma.service.ts`:

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

`src/prisma/prisma.module.ts`:

```ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

---

## Passo 5 — Módulo de Usuários

### Gerar arquivos:

```bash
nest generate resource users
# Escolha: REST API, Yes para CRUD entry points
```

### Arquivos gerados:

### DTO — `src/users/dto/create-user.dto.ts`

```ts
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ description: 'Nome do usuário', example: 'João Silva' })
  name: string;

  @ApiProperty({ description: 'Email do usuário', example: 'joao@example.com' })
  email: string;
}
```

### Service — `src/users/users.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  create(data: CreateUserDto) {
    return this.prisma.user.create({ data });
  }

  findAll() {
    return this.prisma.user.findMany({ orderBy: { createdAt: 'desc' } });
  }

  findOne(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }

  update(id: number, data: Partial<CreateUserDto>) {
    return this.prisma.user.update({ where: { id }, data });
  }

  remove(id: number) {
    return this.prisma.user.delete({ where: { id } });
  }
}
```

### Controller — `src/users/users.controller.ts`

```ts
import { Controller, Get, Post, Put, Delete, Body, Param, ParseIntPipe } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@ApiTags('users')
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @ApiOperation({ summary: 'Criar usuário' })
  @ApiResponse({ status: 201, description: 'Usuário criado com sucesso' })
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }

  @Get()
  @ApiOperation({ summary: 'Listar todos os usuários' })
  @ApiResponse({ status: 200, description: 'Lista de usuários' })
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  @ApiOperation({ summary: 'Buscar usuário por ID' })
  @ApiResponse({ status: 200, description: 'Usuário encontrado' })
  @ApiResponse({ status: 404, description: 'Usuário não encontrado' })
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Atualizar usuário' })
  @ApiResponse({ status: 200, description: 'Usuário atualizado com sucesso' })
  @ApiResponse({ status: 404, description: 'Usuário não encontrado' })
  update(@Param('id', ParseIntPipe) id: number, @Body() dto: Partial<CreateUserDto>) {
    return this.usersService.update(id, dto);
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Deletar usuário' })
  @ApiResponse({ status: 200, description: 'Usuário deletado com sucesso' })
  @ApiResponse({ status: 404, description: 'Usuário não encontrado' })
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

---

## Passo 6 — AppModule e Bootstrap

`src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [PrismaModule, UsersModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`src/main.ts`:

```ts
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Configuração do Swagger
  const config = new DocumentBuilder()
    .setTitle('Minha API')
    .setDescription('API de exemplo com NestJS e Prisma')
    .setVersion('1.0')
    .addTag('users')
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);
  
  await app.listen(3000);
  console.log('Server running on http://localhost:3000');
  console.log('Swagger docs available at http://localhost:3000/api');
}
bootstrap();
```

## Passo 7 — Testando a API

### Acessar Documentação Swagger

Abra seu navegador e acesse: **http://localhost:3000/api**

Aqui você pode:
- Ver todos os endpoints disponíveis
- Testar a API diretamente no navegador
- Ver exemplos de requisições e respostas

### Testando via cURL

Criar usuário:

```bash
curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -d '{"name":"Bruno","email":"bruno@example.com"}'
```

Listar usuários:

```bash
curl http://localhost:3000/users
```

Buscar usuário por ID:

```bash
curl http://localhost:3000/users/1
```

---

## Dicas Importantes

- Centralize a conexão com o banco via `PrismaService`.  
- Use DTOs para organizar dados de entrada e saída.  
- Valide payloads para evitar dados inválidos.  
- Uma API bem escrita é base para qualquer arquitetura escalável.

## Referências

- [Documentação NestJS](https://docs.nestjs.com/)  
- [Documentação Prisma](https://www.prisma.io/docs/)  
- [API com NestJS + Prisma (exemplos)](https://docs.nestjs.com/recipes/prisma)

---

**Fim da Aula 2** — Agora você tem uma **API funcional** e um exemplo prático para evoluir com novos conceitos nas próximas aulas.
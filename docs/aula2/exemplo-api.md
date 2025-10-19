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
nestjs-example/
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

## Passo 1 — Criar o Projeto

```bash
npm i -g @nestjs/cli
nest new nestjs-example
cd nestjs-example

npm install @prisma/client
npm install -D prisma
npm install pg
```

Crie um arquivo `.env` com o conteúdo:

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/nest_example?schema=public"
```

## Passo 2 — Criar o Modelo Prisma

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

## Passo 3 — Criar o PrismaService

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

## Passo 4 — Módulo de Usuários

### DTO — `src/users/dto/create-user.dto.ts`

```ts
export class CreateUserDto {
  name: string;
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
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  @Put(':id')
  update(@Param('id', ParseIntPipe) id: number, @Body() dto: Partial<CreateUserDto>) {
    return this.usersService.update(id, dto);
  }

  @Delete(':id')
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

---

## Passo 5 — AppModule e Bootstrap

`src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [PrismaModule, UsersModule],
})
export class AppModule {}
```

`src/main.ts`:

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
  console.log('Server running on http://localhost:3000');
}
bootstrap();
```

## Passo 6 — Testando a API

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
# Aula 2 — APIs e Arquitetura de Microsserviços
## Slides da Aula

---

## APIs — A Base da Comunicação

### O que são APIs?
- **Ponte entre sistemas** diferentes
- **Contrato bem definido** de comunicação
- Como um **garçom em restaurante**:
  - Você faz pedido (requisição)
  - Garçom leva à cozinha (sistema)
  - Traz comida pronta (resposta)

---

## Tipos de APIs

### REST
- **Mais comum** em aplicações web/mobile
- Usa HTTP (GET, POST, PUT, DELETE)
- **Simples e direto**

### GraphQL
- Cliente pede **exatamente** o que precisa
- **Mais flexível** que REST

### gRPC
- **Muito rápido** e eficiente
- Usado para **serviços falando com serviços**

---

## Componentes de uma API

### Endpoints
```
GET    /api/users       → Lista usuários
POST   /api/users       → Cria usuário
PUT    /api/users/123   → Atualiza usuário
DELETE /api/users/123   → Deleta usuário
```

### Status Codes
- `200 OK` → Sucesso
- `201 Created` → Criado
- `400 Bad Request` → Erro na requisição
- `404 Not Found` → Não encontrado
- `500 Internal Server Error` → Erro servidor

---

## Segurança em APIs

### Métodos de Autenticação
- **API Keys** → Chaves simples
- **OAuth 2.0** → Padrão (Google, GitHub)
- **JWT** → Tokens seguros
- **HTTPS** → Obrigatório para proteção

### Importância
- APIs lidam com **dados sensíveis**
- **Segurança é prioridade**

---

## Documentação e Monitoramento

### Documentação
- **Swagger/OpenAPI** → Padrão para REST
- **Postman/Insomnia** → Ferramentas de teste
- **Exemplos** de requisições e respostas

### Monitoramento
- **Latência** → Tempo de resposta
- **Throughput** → Número de requisições
- **Erros** → Quantos e quais
- **Uptime** → Disponibilidade

---

## Versionamento de APIs

### Estratégias
- **URL**: `/api/v1/users`
- **Header**: `Accept: application/vnd.api+json;version=1`
- **Query**: `/api/users?version=1`

### Importância
- APIs **evoluem**
- **Não quebrar** sistemas existentes

---

## Arquitetura de Microsserviços

### Conceito
- **Vários serviços pequenos** e independentes
- Cada um com **função específica**
- **Comunicação via APIs**

### Exemplo: Loja Online
- **Catálogo** → Produtos
- **Carrinho** → Itens
- **Pagamento** → Transações
- **Usuários** → Autenticação

---

## Características dos Microsserviços

### Principais
- **Desacoplamento** → Serviços independentes
- **Responsabilidade Única** → Uma coisa bem feita
- **Autonomia** → Times e deploys independentes
- **Resiliência** → Falhas isoladas

### Benefícios
- **Escalabilidade independente**
- **Desenvolvimento paralelo**
- **Flexibilidade tecnológica**
- **Tolerância a falhas**

---

## Monólito vs Microsserviços

| Monólito | Microsserviços |
|----------|----------------|
| Tudo em um bloco | Vários serviços |
| Simples de começar | Mais flexível |
| Escalabilidade global | Escalabilidade por serviço |
| Debug simples | Operação complexa |
| Ideal para pequenos | Ideal para grandes |

---

## Desafios dos Microsserviços

### Complexidade
- **Operacional** → Mais infraestrutura
- **Comunicação** → Depende da rede
- **Consistência** → Dados distribuídos
- **Debugging** → Mais difícil rastrear
- **Segurança** → Mais pontos de entrada

### Requisitos
- **Equipes maduras**
- **Processos bem definidos**
- **Infraestrutura robusta**

---

## Padrões Importantes

### Arquiteturais
- **API Gateway** → Ponto único de entrada
- **Service Discovery** → Serviços se encontram
- **Circuit Breaker** → Evita falhas em cascata
- **Database per Service** → Banco por serviço
- **Event-Driven** → Comunicação assíncrona

---

## Ferramentas e Tecnologias

### Containerização
- **Docker, Kubernetes, OpenShift**

### Comunicação
- **REST, GraphQL, gRPC**
- **RabbitMQ, Kafka**

### Monitoramento
- **Prometheus, Grafana, Jaeger**
- **ELK Stack**

### Service Mesh
- **Istio, Linkerd, Consul**

---

## Quando Usar Microsserviços

### ✅ Use quando:
- Aplicação **grande e complexa**
- **Equipes maduras** e estruturadas
- Precisa **escalar partes específicas**
- **Agilidade de deploy** é prioridade

### ❌ Evite quando:
- Projeto **pequeno** ou início
- **Equipe reduzida** e inexperiente
- **Faltam recursos** para infraestrutura

---

## Migração: Monólito → Microsserviços

### Estratégias
- **Strangler Fig Pattern** → Migração gradual
- **Database Decomposition** → Separar bancos
- **API First** → Definir contratos antes

### Processo
1. **Extrair funcionalidades** aos poucos
2. **Separar bancos** compartilhados
3. **Definir APIs** de comunicação

---

## Conclusão

### APIs
- **Base da comunicação** entre sistemas
- **Bem definidas, seguras, documentadas**
- **Facilitam crescimento** e integração

### Microsserviços
- **Flexibilidade e escalabilidade**
- **Exigem maturidade técnica**
- **Não são solução mágica**
- **Poderosos no contexto certo**

---

## Próximos Passos
- Implementação prática com NestJS
- Configuração de banco de dados
- Documentação automática
- Testes e deploy

**Resultado:** Base sólida para arquiteturas modernas!

# API Gateway — O Portão de Entrada Inteligente das Aplicações

## Introdução

Em arquiteturas modernas, especialmente com microsserviços, o **API Gateway** se tornou essencial. Ele funciona como um **intermediário inteligente** que centraliza todas as chamadas de API, simplificando a comunicação entre clientes e serviços de backend.

**O problema:** Com múltiplos microsserviços, clientes precisariam:
- Conhecer o endereço de cada serviço
- Lidar com autenticação em múltiplos lugares
- Coordenar respostas de diferentes serviços

**A solução:** O API Gateway oferece um **único ponto de entrada** que gerencia roteamento, segurança, monitoramento e muito mais.

## O que é um API Gateway?

Um **API Gateway** é uma ferramenta que atua como **intermediário entre clientes e serviços de backend**, apresentando um **único ponto de entrada** para chamadas de API.

### Analogia: Concierge de Hotel

Imagine um **hotel** com vários serviços (restaurante, spa, lavanderia). 

**Sem Concierge:**
- Você precisa ligar para cada serviço separadamente
- Cada um tem seu próprio procedimento
- É confuso e trabalhoso!

**Com Concierge:**
- Você faz **uma única chamada** ao concierge
- Ele **direciona** sua solicitação ao serviço correto
- Ele **agrega informações** de múltiplos serviços se necessário
- Você só precisa conhecer o concierge!

O **API Gateway é o concierge** do seu sistema. Os clientes fazem uma única chamada, e o gateway cuida de tudo: roteamento, segurança, agregação e entrega dos resultados.

## Por que Surgiu a Necessidade?

### O Problema: Microsserviços e Complexidade

**Antes (Monólito):**
```
Cliente → Aplicação Única → Banco de Dados
```
- Simples de gerenciar, mas difícil de escalar

**Depois (Microsserviços):**
```
Cliente → ?
├─ Serviço de Usuários (porta 3001)
├─ Serviço de Produtos (porta 3002)
├─ Serviço de Pagamentos (porta 3003)
└─ ...
```
- Múltiplos serviços, múltiplas portas
- Como o cliente sabe qual porta usar?
- Como gerenciar segurança em cada um?

**Solução: API Gateway**
```
Cliente → API Gateway → [Serviços de Backend]
         (porta única)    (portas internas)
```
- **Um único ponto de entrada** para o cliente
- **Roteamento inteligente** para os serviços corretos
- **Segurança centralizada**

## Como Funciona?

### Fluxo Básico

```
Cliente → API Gateway → Serviço de Backend → Resposta → Cliente
```

**Passos:**
1. Cliente faz requisição ao gateway
2. Gateway **autentica** e **autoriza** o cliente
3. Gateway **aplica rate limiting** (verifica limites)
4. Gateway **roteia** para o serviço correto
5. Serviço processa e retorna resposta
6. Gateway **agrega/transforma** resposta (se necessário)
7. Gateway retorna resposta ao cliente

### Composição de APIs

O gateway pode **combinar respostas de múltiplos serviços** em uma única resposta:

**Sem Gateway:**
```
Cliente faz 3 requisições:
1. GET /pedidos/123
2. GET /produtos?id=1,2,3
3. GET /usuarios/789

3 requisições = 3 latências = lento
```

**Com Gateway:**
```
Cliente faz 1 requisição:
GET /api/pedidos/123

Gateway faz internamente:
1. Busca pedido, produtos e usuário
2. Agrega tudo em uma resposta

1 requisição = rápido e simples
```

## Principais Funcionalidades

### 1. Roteamento de Solicitações

Direciona requisições para os serviços de backend corretos:

```
GET /api/usuarios/*     → Serviço de Usuários
GET /api/produtos/*     → Serviço de Produtos
POST /api/pagamentos/*  → Serviço de Pagamentos
```

### 2. Autenticação e Autorização

Centraliza a segurança:
- **API Keys**: chave simples para identificar cliente
- **OAuth 2.0**: padrão usado por grandes plataformas
- **JWT**: tokens compactos que representam usuário

**Vantagem:** Os serviços de backend não precisam implementar autenticação. Eles confiam no gateway!

### 3. Rate Limiting

Controla quantas requisições um cliente pode fazer:

```
Cliente A (Básico): 100 requisições/hora
Cliente B (Premium): 10.000 requisições/hora
```

**Proteções:**
- Previne abusos
- Protege contra DDoS
- Garante uso justo

### 4. Cache

Armazena respostas para acelerar requisições futuras:

```
Requisição 1: Cliente → Gateway → Serviço → Cache
Requisição 2: Cliente → Gateway → Cache (muito mais rápido!)
```

### 5. Monitoramento e Logging

Coleta dados sobre todas as requisições:
- **Métricas**: latência, throughput, taxa de erro
- **Logs**: histórico completo de uso
- **Dashboards**: visualização em tempo real

### 6. Balanceamento de Carga

Distribui requisições entre múltiplas instâncias:

```
API Gateway
    ├─ Serviço (instância 1) - 50% do tráfego
    ├─ Serviço (instância 2) - 50% do tráfego
    └─ Serviço (instância 3) - reserva
```

## Benefícios

1. **Simplificação**: Cliente só conhece o gateway, não os serviços internos
2. **Segurança Centralizada**: Implementada uma vez, aplicada em todos os serviços
3. **Escalabilidade**: Facilita balanceamento de carga e distribuição de tráfego
4. **Observabilidade**: Visibilidade completa do sistema (métricas e logs centralizados)
5. **Flexibilidade**: Facilita migração de serviços sem afetar clientes

## Desafios e Considerações

### 1. Ponto Único de Falha

**Problema:** Se o gateway cair, todo o sistema fica inacessível.

**Mitigação:**
- Múltiplas instâncias do gateway
- Load balancer na frente
- Health checks para detectar problemas

### 2. Escalabilidade

**Problema:** Pode se tornar gargalo se não escalado corretamente.

**Mitigação:**
- Escalar horizontalmente (adicionar instâncias)
- Cache agressivo
- Monitoramento para identificar gargalos

### 3. Complexidade Operacional

**Problema:** O gateway também precisa ser monitorado e gerenciado.

**Mitigação:**
- Automação e ferramentas de gerenciamento
- Equipe treinada
- Documentação atualizada

## Exemplo Prático: E-commerce

**Cenário:** Cliente precisa visualizar informações de um pedido.

**Sem Gateway:**
```javascript
// Cliente precisa fazer 4 requisições
const pedido = await fetch('http://pedidos:3001/pedidos/123');
const produtos = await fetch('http://produtos:3002/produtos?id=1,2,3');
const usuario = await fetch('http://usuarios:3003/usuarios/789');
const pagamento = await fetch('http://pagamentos:3004/pagamentos/pedido-123');

// Agregar tudo manualmente
const resultado = {
  pedido: pedido.data,
  produtos: produtos.data,
  usuario: usuario.data,
  pagamento: pagamento.data
};
```

**Problemas:**
- Cliente precisa conhecer 4 endereços diferentes
- 4 requisições = 4 latências = experiência lenta

**Com Gateway:**
```javascript
// Cliente faz uma única requisição
const resposta = await fetch('http://gateway/api/pedidos/123');

// Gateway faz internamente:
// 1. Busca pedido, produtos, usuário e pagamento
// 2. Agrega tudo em uma resposta única

const resultado = resposta.data; // Tudo pronto!
```

**Vantagens:**
- Cliente conhece apenas o gateway
- 1 requisição = experiência rápida
- Gateway centraliza mudanças

## Ferramentas Populares

### Kong
- **Open source** e gratuito (versão Community)
- Altamente extensível com plugins
- Ideal para controle total

### AWS API Gateway
- **Gerenciado** pela AWS (serverless)
- Escalável automaticamente
- Pay-per-use

### NGINX
- **Web server** e reverse proxy
- Muito performático
- Open source

### Apache APISIX
- **Open source** e gratuito
- Cloud-native (Kubernetes)
- Alta performance

## Quando Usar ou Não Usar

### ✅ Use quando:

- **Microsserviços**: múltiplos serviços precisam ser gerenciados
- **Segurança centralizada**: quer unificar autenticação/autorização
- **Múltiplos clientes**: web, mobile, terceiros
- **Agregação de dados**: precisa combinar respostas de múltiplos serviços
- **Rate limiting**: precisa controlar uso por cliente

### ❌ Evite quando:

- **Aplicação simples**: sistema pequeno com poucos endpoints
- **Monólito**: gateway pode ser overkill
- **Equipe pequena**: falta recursos para gerenciar
- **Serviço único**: pode ser desnecessário

### Gateway Gerenciado vs Self-Hosted

| Critério | Gerenciado (AWS) | Self-Hosted (Kong) |
|----------|------------------|-------------------|
| **Setup** | Rápido | Requer configuração |
| **Custo** | Pay-per-use | Infraestrutura própria |
| **Flexibilidade** | Limitada | Total controle |
| **Ideal para** | Começar rápido | Controle total |

## Arquitetura Básica

```
┌─────────────────────────────────┐
│         Clientes                │
│  (Web, Mobile, Terceiros)       │
└──────────────┬───────────────────┘
               │
               │ HTTPS
               ▼
┌─────────────────────────────────┐
│         API Gateway             │
│  - Autenticação                 │
│  - Rate Limiting                 │
│  - Roteamento                    │
│  - Cache                         │
└──────┬───────────────────────────┘
       │
       ├─────────┬─────────┬─────────┐
       ▼         ▼         ▼         ▼
   ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
   │Serviço│ │Serviço│ │Serviço│ │Serviço│
   │   1   │ │   2   │ │   3   │ │   4   │
   └───────┘ └───────┘ └───────┘ └───────┘
```

## Conclusão

O **API Gateway** é um componente essencial em arquiteturas modernas, especialmente com microsserviços. Ele atua como um **ponto central de controle**, facilitando a comunicação entre clientes e serviços de backend.

### Principais Takeaways:

1. **Simplificação**: Um único ponto de entrada
2. **Segurança**: Centralização facilita implementação
3. **Escalabilidade**: Facilita crescimento e distribuição de carga
4. **Observabilidade**: Visibilidade completa do sistema
5. **Flexibilidade**: Facilita mudanças sem afetar clientes

### Princípio Central:

> **"O API Gateway é como o concierge de um hotel: você só precisa se comunicar com ele, e ele cuida de direcionar, agregar e entregar tudo que você precisa, de forma organizada e eficiente."**

---

## Referências

- [IBM - O que é um API Gateway?](https://www.ibm.com/br-pt/think/topics/api-gateway)
- [Kong - What is an API Gateway? Core Fundamentals and Use Cases](https://konghq.com/blog/learning-center/what-is-an-api-gateway)

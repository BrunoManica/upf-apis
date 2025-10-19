# APIs — A Base da Comunicação entre Sistemas

## Introdução

Hoje, praticamente todos os sistemas conversam entre si. Quando você faz login com uma conta de rede social, rastreia um pedido ou paga uma compra online, há uma comunicação acontecendo por trás dos bastidores.  
Essa comunicação é feita por **APIs**, que funcionam como **pontes bem organizadas entre aplicações** diferentes. Elas definem **como** os sistemas devem trocar informações de forma segura e padronizada.

## O que são APIs?

Uma API (Application Programming Interface) é como um **garçom em um restaurante**:
- Você faz um pedido (requisição)
- O garçom leva até a cozinha (sistema)
- Ele traz de volta a comida pronta (resposta)

A cozinha não precisa saber quem você é, e você não precisa saber como a comida é feita. Esse **contrato bem definido** é o que torna as APIs tão poderosas: cada parte sabe exatamente o que esperar da outra.

## Tipos de APIs

Existem vários estilos de API, cada um com seu propósito:

- **REST**: o mais comum. Usa HTTP e métodos como GET, POST, PUT e DELETE. É simples, direto e muito usado em aplicações web e mobile.  
- **GraphQL**: permite que o cliente peça **exatamente** os dados que precisa, de forma mais flexível que REST.  
- **gRPC**: muito rápido e eficiente, usado principalmente quando **serviços falam com outros serviços**, como em sistemas de alta performance.

## Componentes Básicos de uma API

- **Endpoints**: são os endereços (URLs) que representam recursos.  
  Exemplo:  
  ```
  GET    /api/users       → Lista usuários  
  POST   /api/users       → Cria usuário  
  PUT    /api/users/123   → Atualiza usuário 123  
  DELETE /api/users/123   → Deleta usuário 123
  ```

- **Métodos HTTP**: indicam a ação (GET = ler, POST = criar, PUT = atualizar, DELETE = remover).

- **Status Codes**: informam o resultado da requisição:  
  - `200 OK` → deu tudo certo  
  - `201 Created` → recurso criado  
  - `400 Bad Request` → problema na requisição  
  - `404 Not Found` → não encontrado  
  - `500 Internal Server Error` → erro no servidor

## Segurança em APIs

APIs lidam com dados sensíveis — por isso, **segurança é prioridade**.

- **API Keys**: chaves simples para identificar quem faz a requisição.  
- **OAuth 2.0**: padrão de autorização usado por grandes plataformas (como Google e GitHub).  
- **JWT**: tokens compactos e seguros que representam um usuário sem necessidade de sessão no servidor.  
- **HTTPS**: obrigatório para proteger os dados durante a transmissão.

## Documentação

Uma API sem documentação é praticamente inútil. Os desenvolvedores precisam saber **como usar**.

- **Swagger/OpenAPI**: padrão mais usado para documentar APIs REST.  
- **Postman** e **Insomnia**: ferramentas para testar e explorar APIs.  

Uma boa documentação inclui endpoints, exemplos de requisições e respostas, códigos de erro e guias rápidos.

## Versionamento

APIs evoluem — e é importante fazer isso **sem quebrar os sistemas que já dependem delas**.

Exemplos de versionamento:
- Na URL: `/api/v1/users`
- No header: `Accept: application/vnd.api+json;version=1`
- Na query: `/api/users?version=1`

## Monitoramento e Limites de Uso

Manter uma API saudável significa **saber o que está acontecendo** com ela:
- **Latência** → tempo de resposta
- **Throughput** → número de requisições
- **Erros** → quantos e quais
- **Uptime** → disponibilidade

Ferramentas como Prometheus, Grafana e Datadog ajudam a acompanhar tudo isso.

Além disso, técnicas de **Rate Limiting** impedem abusos (por exemplo, limitar 1000 requisições por hora por cliente), garantindo estabilidade e uso justo.

## Conclusão

APIs são a **cola que conecta sistemas modernos**. Uma API bem feita é:
- Bem definida
- Segura
- Documentada
- Monitorada

E, mais importante: **facilita o crescimento de aplicações** e a integração entre serviços — base essencial para arquiteturas modernas, como microsserviços.

## Referências

- [Red Hat - What are APIs](https://www.redhat.com/pt-br/topics/api/what-are-application-programming-interfaces)
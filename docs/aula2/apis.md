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

- **REST**: estilo arquitetural que normalmente usa HTTP, recursos e métodos como GET, POST, PUT e DELETE. É muito usado em aplicações web e mobile.  
- **SOAP**: protocolo de comunicação baseado em mensagens estruturadas, normalmente escritas em XML. É comum em integrações corporativas que exigem contratos rígidos e padrões adicionais de segurança e transação.  
- **GraphQL**: permite que o cliente peça **exatamente** os dados que precisa, de forma mais flexível que REST.  
- **gRPC**: muito rápido e eficiente, usado principalmente quando **serviços falam com outros serviços**, como em sistemas de alta performance.

## SOAP, XML, REST e JSON

Esses termos aparecem juntos com frequência, mas não representam a mesma categoria de tecnologia:

| Termo | O que é | Uso comum |
|---|---|---|
| SOAP | Protocolo de comunicação | Integrações corporativas com contrato rígido |
| REST | Estilo arquitetural | APIs web organizadas em recursos |
| XML | Formato textual de dados | Mensagens SOAP e integrações que exigem documentos estruturados |
| JSON | Formato textual de dados | Requisições e respostas de APIs REST |

SOAP e REST definem maneiras diferentes de organizar a comunicação. XML e JSON definem como os dados são representados durante essa comunicação.

### SOAP e XML

Uma mensagem SOAP é um documento XML com uma estrutura chamada envelope. O envelope identifica a mensagem SOAP, o cabeçalho pode carregar informações adicionais e o corpo contém os dados da operação.

Exemplo simplificado de uma requisição para buscar um usuário:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
        <buscarUsuario>
            <id>123</id>
        </buscarUsuario>
    </soap:Body>
</soap:Envelope>
```

Em uma API SOAP, as operações e os tipos de dados podem ser descritos por um documento WSDL. Esse contrato permite que ferramentas gerem parte do código necessário para clientes e servidores. Em contrapartida, as mensagens XML costumam ser mais extensas e o contrato mais rígido.

SOAP pode ser adequado quando uma integração precisa seguir padrões corporativos já estabelecidos, contratos formais ou mecanismos da família WS-*, como WS-Security. Isso não significa que toda API corporativa deva usar SOAP; a escolha depende dos requisitos e dos sistemas envolvidos.

### REST e JSON

Em uma API REST, o foco está nos recursos. Um usuário pode ser representado pelo caminho `/api/users/123`, e o método HTTP informa a operação desejada.

```http
GET /api/users/123 HTTP/1.1
Host: localhost:8080
Accept: application/json
```

Uma resposta em JSON poderia ser:

```json
{
  "id": 123,
  "name": "Ana Silva",
  "email": "ana@email.com"
}
```

JSON representa objetos por pares de chave e valor e listas por colchetes. Ele costuma ser mais compacto que XML e possui integração direta com JavaScript, por isso se tornou o formato mais comum em APIs web.

REST não exige JSON. O mesmo recurso poderia ser devolvido em XML se o cliente solicitasse `application/xml` e o servidor oferecesse esse formato:

```xml
<user>
    <id>123</id>
    <name>Ana Silva</name>
    <email>ana@email.com</email>
</user>
```

O cabeçalho `Content-Type` informa o formato do corpo enviado. O cabeçalho `Accept` informa qual formato o cliente deseja receber. Quando trabalharmos com Spring Boot, usaremos principalmente `application/json`.

### Comparação prática

| Aspecto | SOAP com XML | REST com JSON |
|---|---|---|
| Organização | Operações definidas por um protocolo | Recursos manipulados por métodos HTTP |
| Contrato | Frequentemente descrito por WSDL | Frequentemente descrito por OpenAPI |
| Formato mais comum | XML | JSON |
| Tamanho das mensagens | Geralmente maior | Geralmente menor |
| Flexibilidade | Contrato mais rígido | Contrato geralmente mais simples de evoluir |
| Cenário frequente | Sistemas corporativos e legados | Aplicações web, mobile e microsserviços |

Não existe vencedor universal. Em um sistema novo com clientes web e mobile, REST com JSON costuma ser uma escolha simples. Quando é necessário integrar com um serviço SOAP existente, o cliente deve respeitar o contrato XML definido por esse serviço.

## Componentes Básicos de uma API

- **Endpoints**: são os endereços (URLs) que representam recursos.  
  Exemplo:  
```bash
  GET    /api/users       → Lista usuários  
  POST   /api/users       → Cria usuário  
  PUT    /api/users/123   → Atualiza usuário 123  
  DELETE /api/users/123   → Deleta usuário 123
```

- **Métodos HTTP**: indicam a ação (GET = ler, POST = criar, PUT = atualizar, DELETE = remover).

- **Status Codes**: informam o resultado da requisição:  
```bash
  - `200 OK` → deu tudo certo  
  - `201 Created` → recurso criado  
  - `400 Bad Request` → problema na requisição  
  - `404 Not Found` → não encontrado  
  - `500 Internal Server Error` → erro no servidor
```

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
```bash
- Na URL: `/api/v1/users`
- No header: `Accept: application/vnd.api+json;version=1`
- Na query: `/api/users?version=1`
```

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

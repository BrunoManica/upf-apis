# Revisão Geral das Aulas 1 a 6

## Objetivo da aula

Esta revisão serve para juntar as ideias principais das seis primeiras aulas.

Até aqui, vimos várias peças separadas:

* containers e Docker;
* APIs REST;
* Spring Boot com MongoDB;
* organização de código com camadas;
* princípios SOLID;
* API Gateway;
* mensageria com RabbitMQ;
* observabilidade com logs, métricas, health checks e Prometheus.

Agora faz sentido olhar para tudo isso como um conjunto.

O objetivo não é decorar nomes de ferramentas. O objetivo é entender onde cada peça entra quando estamos construindo APIs e microsserviços em Java com Spring Boot.

## Resultado final

Ao final desta revisão, você deve conseguir explicar este desenho:

```text
Cliente
  |
  v
API Gateway
  |
  +--> users-service
  |
  +--> products-service
  |
  +--> orders-service
          |
          +--> MongoDB
          |
          +--> RabbitMQ
                  |
                  +--> notifications-service

Todos os serviços:
  - rodam em containers
  - expõem APIs REST quando necessário
  - usam camadas internas bem separadas
  - registram logs
  - expõem health checks e métricas
```

Esse desenho resume boa parte do caminho feito até aqui.

Não é um sistema completo de produção, mas já mostra a base de uma arquitetura moderna:

* cada serviço tem uma responsabilidade;
* o gateway organiza a entrada;
* o banco guarda dados do domínio;
* o message broker entrega eventos;
* a observabilidade ajuda a entender o que está acontecendo.

## Contexto

Quando começamos a estudar desenvolvimento backend, é comum pensar só no endpoint.

Por exemplo:

```text
POST /api/v1/orders
```

Mas uma API real não é só uma rota.

Por trás de uma rota existem várias decisões:

* onde a aplicação roda;
* como ela conversa com o banco;
* como o código fica organizado;
* como outros serviços chamam essa API;
* como a API avisa outros sistemas quando algo acontece;
* como a equipe descobre problemas em produção.

As aulas 1 a 6 foram montadas para responder essas perguntas aos poucos.

Primeiro entendemos onde a aplicação roda. Depois vimos como criar APIs. Em seguida, organizamos melhor o código. Depois colocamos gateway na frente, mensageria entre serviços e observabilidade para acompanhar tudo.

## Explicação conceitual

### A ideia central

Uma aplicação backend não vive isolada.

Ela normalmente faz parte de um ambiente com banco de dados, outras APIs, filas, containers, ferramentas de monitoramento e clientes externos.

Por isso, aprender microsserviços não é só aprender a criar vários projetos Spring Boot. É entender comunicação, responsabilidade, isolamento, falha e operação.

### O caminho das aulas

A Aula 1 apresentou containers e Docker.

Antes de falar em microsserviços, precisamos entender como uma aplicação pode rodar de forma mais previsível. O container empacota a aplicação e suas dependências em um ambiente controlado. Isso ajuda a evitar o famoso problema de "na minha máquina funciona".

A Aula 2 entrou em APIs e microsserviços.

Aqui vimos que uma API é um contrato de comunicação. Ela define como outro sistema pode chamar nossa aplicação. Também começamos a separar controller, service, repository, model e DTO.

A Aula 3 trouxe SOLID.

SOLID não existe para deixar o código bonito. Ele ajuda quando o código começa a crescer. A ideia é evitar classes que fazem coisa demais, dependências difíceis de trocar e mudanças pequenas que quebram muitas partes do sistema.

A Aula 4 mostrou API Gateway.

Quando temos vários serviços, não queremos que o cliente conheça todos os endereços internos. O gateway vira uma porta de entrada. Ele roteia chamadas e pode concentrar regras transversais, como autenticação, limite de chamadas e cache.

A Aula 5 trouxe mensageria.

Nem tudo precisa acontecer durante a requisição HTTP. Quando um pedido é criado, por exemplo, a API pode salvar o pedido e publicar um evento. Outros serviços consomem esse evento depois. Isso reduz acoplamento e melhora a evolução do sistema.

A Aula 6 trouxe observabilidade.

Quando temos várias partes trabalhando juntas, precisamos enxergar o sistema. Logs, métricas, health checks e Prometheus ajudam a investigar problemas, medir comportamento e saber se a aplicação está saudável.

### Como essas peças se conectam

Pense em uma venda online.

O cliente chama o gateway.

O gateway encaminha para o serviço de pedidos.

O serviço de pedidos recebe a requisição no controller.

O controller valida a entrada e chama o service.

O service aplica a regra de negócio.

O repository salva o pedido no MongoDB.

Depois disso, o serviço publica um evento `pedido.criado` no RabbitMQ.

Um serviço de notificações consome esse evento e envia uma mensagem para o cliente.

Enquanto tudo isso acontece, a aplicação registra logs, expõe health check e gera métricas para o Prometheus.

Esse é o ponto principal da revisão: cada aula explicou uma parte desse cenário.

## Setup inicial

Esta revisão não precisa de instalação nem criação de projeto.

Use este material para estudar antes de uma prática, prova, apresentação ou continuação do projeto.

Se quiser revisar com mais profundidade, mantenha por perto:

* o material da Aula 1 sobre Docker;
* a API REST com Spring Boot e MongoDB da Aula 2;
* o exemplo de SOLID da Aula 3;
* o gateway da Aula 4;
* o projeto com RabbitMQ da Aula 5;
* a aula e o tutorial de observabilidade da Aula 6.

## Passo a passo

### 1. Revise a Aula 1: containers, VMs e Docker

Comece lembrando a diferença entre máquina virtual e container.

Uma máquina virtual simula uma máquina inteira, incluindo sistema operacional próprio.

Um container isola a aplicação e suas dependências, mas compartilha o kernel do sistema operacional.

Na prática, usamos Docker porque ele facilita:

* rodar MongoDB localmente;
* subir RabbitMQ com interface web;
* empacotar aplicações;
* criar ambientes parecidos entre máquinas diferentes;
* usar Docker Compose para subir vários serviços juntos.

O ponto que você precisa guardar:

```text
Docker ajuda a rodar aplicações e dependências de forma previsível.
```

### 2. Revise a Aula 2: APIs REST e microsserviços

API é uma forma padronizada de comunicação entre sistemas.

Em uma API REST, usamos HTTP de forma organizada:

* `GET` para leitura;
* `POST` para criação;
* `PUT` para atualização completa;
* `PATCH` para atualização parcial;
* `DELETE` para remoção.

Também usamos status HTTP para comunicar o resultado:

* `200 OK`;
* `201 Created`;
* `204 No Content`;
* `400 Bad Request`;
* `404 Not Found`;
* `409 Conflict`.

Na prática com Spring Boot, a estrutura mínima profissional ficou assim:

```text
controller -> service -> repository -> MongoDB
       DTO -> mapper -> model
```

O controller recebe HTTP.

O service concentra regra de negócio.

O repository acessa o MongoDB.

O model representa o documento salvo.

O DTO representa entrada ou saída da API.

O mapper converte model para DTO.

O ponto que você precisa guardar:

```text
API boa tem contrato claro e responsabilidade bem separada no código.
```

### 3. Revise a Aula 3: SOLID

SOLID é um conjunto de princípios para reduzir bagunça quando o sistema cresce.

O mais importante para este momento é entender a intenção:

* uma classe não deve fazer tudo;
* uma mudança pequena não deveria quebrar várias partes;
* regras diferentes podem ser separadas em classes diferentes;
* depender de abstrações pode facilitar troca de implementação quando existe motivo real para isso.

Em uma API Spring Boot, isso aparece de forma simples:

* controller fino;
* service com regra;
* repository isolando banco;
* mapper isolando conversão;
* DTO isolando contrato externo.

O ponto que você precisa guardar:

```text
SOLID ajuda a manter o código modificável sem virar arquitetura exagerada.
```

### 4. Revise a Aula 4: API Gateway

Quando temos um serviço só, o cliente chama esse serviço diretamente.

Quando temos vários serviços, isso começa a complicar.

Sem gateway, o cliente precisa conhecer vários endereços:

```text
http://localhost:8081/users
http://localhost:8082/products
http://localhost:8083/orders
```

Com gateway, o cliente conhece uma entrada principal:

```text
http://localhost:8080
```

O gateway recebe a chamada e encaminha para o serviço correto.

Ele pode cuidar de regras transversais, como:

* autenticação;
* rate limiting;
* cache;
* roteamento;
* padronização da entrada.

Mas existe um cuidado importante: gateway não é lugar para regra de negócio.

Regra de usuário fica no serviço de usuários.

Regra de produto fica no serviço de produtos.

Regra de pedido fica no serviço de pedidos.

O ponto que você precisa guardar:

```text
Gateway organiza a entrada, mas não substitui os serviços de negócio.
```

### 5. Revise a Aula 5: mensageria com RabbitMQ

HTTP é ótimo quando precisamos de resposta imediata.

Mensageria é útil quando um serviço precisa avisar que algo aconteceu, sem depender diretamente de quem vai reagir.

Um exemplo clássico:

```text
orders-service publica pedido.criado
notifications-service consome pedido.criado
```

O serviço de pedidos não precisa chamar diretamente o serviço de notificações.

Ele publica um evento.

O RabbitMQ entrega esse evento para quem estiver interessado.

Os conceitos principais são:

* producer: quem publica a mensagem;
* consumer: quem consome a mensagem;
* queue: fila onde a mensagem fica;
* exchange: ponto de entrada da mensagem no RabbitMQ;
* routing key: chave usada para rotear a mensagem;
* binding: ligação entre exchange e fila.

Também vimos a diferença entre evento e RPC.

Evento é usado quando algo aconteceu:

```text
pedido.criado
```

RPC é usado quando um serviço faz uma pergunta e espera resposta:

```text
existe estoque para este produto?
```

O ponto que você precisa guardar:

```text
Mensageria reduz acoplamento quando um serviço precisa avisar outros serviços.
```

### 6. Revise a Aula 6: observabilidade

Quando o sistema tem várias partes, precisamos enxergar o que está acontecendo.

Monitoramento ajuda a perceber sintomas:

* a API está fora do ar;
* a latência subiu;
* os erros aumentaram;
* a memória está alta.

Observabilidade ajuda a investigar causas:

* onde a requisição demorou?
* qual serviço falhou?
* qual endpoint está com erro?
* o MongoDB está saudável?
* a fila acumulou mensagens?

Os três sinais mais importantes são:

* logs;
* métricas;
* traces.

Na prática com Spring Boot, começamos com:

* logs usando SLF4J;
* health check com Spring Boot Actuator;
* métricas com Micrometer;
* coleta com Prometheus.

O ponto que você precisa guardar:

```text
Observabilidade existe para trocar chute por investigação baseada em dados.
```

### 7. Monte a visão completa

Agora junte tudo.

Uma arquitetura básica de microsserviços precisa pensar em:

```text
Execução local       -> Docker e Docker Compose
Entrada HTTP         -> API REST
Código interno       -> controller, service, repository, model, DTO e mapper
Organização          -> SOLID sem exagero
Entrada única        -> API Gateway
Comunicação assíncrona -> RabbitMQ
Operação             -> logs, health checks, métricas e Prometheus
```

Cada tecnologia resolve um problema diferente.

O erro comum é tentar usar tudo ao mesmo tempo sem saber por quê.

O caminho certo é perguntar:

```text
Qual problema eu estou tentando resolver?
```

Depois disso, a ferramenta faz mais sentido.

### 8. Perguntas para revisar antes da prova ou prática

Responda com suas palavras:

1. Qual é a diferença entre container e máquina virtual?
2. Por que usamos Docker Compose em aulas com MongoDB e RabbitMQ?
3. Qual é o papel do controller em uma API Spring Boot?
4. Por que regra de negócio deve ficar no service?
5. Por que DTO não deve ser misturado com model?
6. O que o repository faz em uma aplicação com Spring Data MongoDB?
7. Qual problema o SOLID tenta reduzir?
8. Por que um gateway não deve conter regra de negócio?
9. Quando faz sentido usar RabbitMQ em vez de chamar outro serviço por HTTP?
10. Qual é a diferença entre evento e RPC?
11. O que um health check mostra?
12. Qual é a diferença entre log e métrica?
13. Para que serve o Prometheus?
14. Por que observabilidade fica mais importante em microsserviços?

### 9. Checklist mental de uma API bem organizada

Antes de considerar uma API pronta, confira:

* as rotas usam recursos no plural;
* os métodos HTTP fazem sentido;
* os status HTTP estão corretos;
* o controller não tem regra de negócio;
* o service concentra a regra;
* o repository acessa o banco;
* os DTOs ficam em arquivos próprios;
* o mapper converte model para response;
* o MongoDB roda em Docker;
* a aplicação usa variável de ambiente para conexão;
* existe health check;
* existem logs úteis;
* existem métricas básicas.

## Código completo

Esta revisão não tem código completo novo.

O código completo está distribuído nos tutoriais das aulas anteriores:

* API REST com Spring Boot e MongoDB;
* Payment API com organização melhorada e princípios SOLID;
* API Gateway com Spring Cloud Gateway;
* mensageria com RabbitMQ;
* observabilidade com Actuator, Micrometer e Prometheus.

O papel desta página é conectar as ideias, não criar mais um projeto.

## Erros comuns

### Decorar ferramenta sem entender o problema

Docker, Gateway, RabbitMQ e Prometheus são ferramentas.

Antes de usar qualquer uma delas, pergunte qual problema ela resolve.

### Misturar responsabilidades no código

Um erro comum é colocar tudo no controller.

No começo parece mais rápido, mas depois o código fica difícil de testar, alterar e entender.

### Criar microsserviços cedo demais

Microsserviços ajudam quando existe necessidade real de separação, escala ou autonomia.

Se o sistema ainda é pequeno e a equipe ainda está aprendendo o básico, começar com uma API bem organizada pode ser melhor.

### Usar mensageria para tudo

RabbitMQ não substitui HTTP.

Use HTTP quando precisar de resposta imediata.

Use evento quando quiser avisar que algo aconteceu.

Use RPC com cuidado, quando realmente fizer sentido perguntar algo por mensageria e esperar uma resposta.

### Ignorar observabilidade

Uma API sem logs, health check e métricas fica difícil de operar.

Enquanto tudo está na máquina local, parece simples.

Em produção, sem observabilidade, a equipe descobre problemas tarde e investiga com pouca informação.

### Exagerar na arquitetura

Organização é importante, mas exagero atrapalha.

Para aulas iniciais, prefira uma separação clara e simples:

```text
controller
service
repository
model
dto
mapper
```

Isso já é suficiente para construir uma base profissional sem transformar a aula em arquitetura avançada.

## Resumo

As aulas 1 a 6 montam a base para construir APIs e microsserviços com Java 17 e Spring Boot.

Docker ajuda a rodar aplicações e dependências.

APIs REST organizam a comunicação HTTP.

Spring Boot permite construir serviços com controller, service, repository, model, DTO e mapper.

SOLID ajuda a manter o código mais fácil de evoluir.

API Gateway organiza a entrada em sistemas com vários serviços.

RabbitMQ permite comunicação assíncrona por eventos.

Observabilidade ajuda a entender o sistema em execução usando logs, métricas, health checks e Prometheus.

O mais importante é conectar ferramenta com problema.

Quando você entende o problema, a tecnologia deixa de parecer uma lista de nomes e passa a virar uma escolha técnica consciente.

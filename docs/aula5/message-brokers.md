# Message Brokers, filas e eventos com RabbitMQ e Kafka

## Objetivo da aula

Vamos entender o que é um message broker e por que ele aparece quando um microsserviço precisa avisar outro sem ficar esperando uma resposta HTTP.

## Resultado final

A ideia principal da aula é esta:

```text
Pedidos API
   |
   | publica evento pedido.criado
   v
Message Broker
   |
   +--> Notificações Service
   |
   +--> Faturamento Service
   |
   +--> Analytics Service
```

A API de pedidos cria o pedido e publica um evento.

Outros serviços recebem esse evento depois, cada um no seu tempo.

Na prática, esse broker poderia ser RabbitMQ, Kafka, ActiveMQ, Amazon SQS, Google Pub/Sub ou outra ferramenta de mensageria. Cada uma tem detalhes próprios, mas a conversa começa pelo mesmo ponto: separar quem produz uma mensagem de quem consome essa mensagem.

## Contexto

Até aqui, quando uma aplicação precisava conversar com outra, usamos HTTP:

```text
Cliente -> API Gateway -> Serviço de Usuários
```

Isso funciona muito bem para perguntas diretas:

```text
Me devolva o usuário 10.
Crie este produto.
Atualize este cadastro.
```

Mas nem toda comunicação precisa acontecer na hora.

Imagine uma compra em uma loja virtual. Quando o pedido é criado, várias coisas podem acontecer:

* enviar e-mail para o cliente;
* baixar estoque;
* registrar uma métrica;
* iniciar faturamento;
* avisar um sistema externo;
* alimentar um painel de vendas.

Se a API de pedidos chamar tudo isso diretamente por HTTP, o cliente fica esperando cada etapa terminar.

Também aparece outro problema: a API de pedidos passa a conhecer serviços demais. Ela precisa saber onde está o serviço de e-mail, como chamar o estoque, qual endpoint usar no faturamento e o que fazer quando cada integração falhar.

Se o serviço de e-mail estiver fora do ar, a criação do pedido pode falhar mesmo com o pedido já estando válido.

É nesse ponto que a mensageria começa a fazer sentido.

## Explicação conceitual

Um message broker é uma aplicação intermediária que recebe mensagens, armazena essas mensagens por algum tempo e entrega para outros sistemas.

Pense nele como uma central de distribuição. A API de pedidos não precisa saber exatamente quem vai reagir ao evento. Ela só diz:

```text
pedido.criado aconteceu
```

O broker recebe essa mensagem e entrega para quem estiver interessado.

Por baixo dos panos, o broker ajuda em três pontos importantes:

* desacoplar producer e consumer;
* permitir comunicação assíncrona;
* controlar entrega, armazenamento, roteamento e reprocessamento de mensagens.

### Producer

Producer é quem envia a mensagem.

No nosso exemplo, o producer é a API de pedidos:

```text
Pedidos API -> Message Broker
```

Ela cria o pedido, salva no banco e publica um evento dizendo que o pedido foi criado.

O producer não deve conhecer a regra interna dos outros serviços. Ele não deve pensar assim:

```text
envie um e-mail
gere uma nota
atualize um dashboard
```

Ele deve publicar um fato do negócio:

```text
pedido.criado
```

Isso deixa o sistema mais flexível. Se amanhã entrar um serviço antifraude interessado nesse evento, a API de pedidos não precisa ser reescrita para conhecer esse novo serviço.

### Consumer

Consumer é quem recebe e processa a mensagem.

No nosso exemplo, vários serviços podem consumir o mesmo evento:

```text
Message Broker -> Notificações Service
Message Broker -> Faturamento Service
Message Broker -> Analytics Service
```

Cada consumer cuida da sua responsabilidade.

O serviço de notificações envia e-mail. O serviço de faturamento inicia a cobrança. O serviço de analytics atualiza indicadores.

O producer não precisa esperar tudo isso terminar para responder ao cliente.

### Evento

Evento é uma mensagem dizendo que algo importante já aconteceu.

Um bom nome de evento costuma ficar no passado:

```text
pedido.criado
pagamento.aprovado
usuario.cadastrado
produto.atualizado
```

Isso muda a forma de pensar.

O producer não está mandando o consumer executar uma ordem direta. Ele está avisando que um fato aconteceu.

Essa diferença parece pequena, mas ajuda bastante no desenho de microsserviços. Quando você publica um evento, outros serviços podem reagir sem que o producer conheça esses serviços.

### Fila

Fila é um lugar onde mensagens ficam aguardando processamento.

Ela ajuda quando existe um trabalho que precisa ser feito, mas não precisa ser feito exatamente no mesmo instante da requisição HTTP.

Exemplo:

```text
pedido.criado -> fila de notificações -> notifications-service
```

Se o serviço de notificações estiver desligado, a mensagem pode ficar aguardando. Quando o serviço voltar, ele continua consumindo.

Esse modelo combina bem com processamento assíncrono, filas de trabalho e tarefas que precisam ser distribuídas entre uma ou mais instâncias de um consumer.

### Tópico

Tópico também recebe mensagens, mas costuma ser usado quando vários consumidores podem se interessar pelo mesmo tipo de evento.

Exemplo:

```text
tópico pedidos
   |
   +--> grupo notificações
   |
   +--> grupo faturamento
   |
   +--> grupo analytics
```

No Kafka, tópico é uma ideia central. O producer publica eventos em um tópico, e consumers leem esses eventos.

Uma diferença importante é que Kafka costuma manter os eventos por um período configurado. Isso permite que um consumer leia eventos antigos, reprocesse informações ou comece a consumir a partir de um ponto específico.

Em uma fila tradicional, é comum a mensagem sair da fila depois que o consumer confirma o processamento. Em Kafka, o evento fica no log por um tempo, e cada consumer controla até onde já leu.

### ACK

ACK vem de acknowledgement, ou confirmação.

É a forma de o consumer dizer ao broker:

```text
processei essa mensagem com sucesso
```

Isso importa porque uma mensagem pode chegar ao consumer, mas o serviço pode cair antes de terminar o processamento.

Sem confirmação, o broker não tem como saber se deu certo.

Em RabbitMQ, é comum falar em confirmação da mensagem consumida. Em Kafka, é comum falar em offset, que representa a posição até onde um consumer já leu dentro de um tópico.

São mecanismos diferentes, mas resolvem uma preocupação parecida: controlar o avanço do consumo.

### RabbitMQ

RabbitMQ é um message broker muito usado para filas, roteamento e processamento assíncrono.

Ele trabalha muito bem quando você quer:

* criar filas de trabalho;
* distribuir mensagens entre consumers;
* usar roteamento por regras;
* separar producer e consumer com uma configuração simples;
* trabalhar com padrões como direct, fanout e topic exchange.

No RabbitMQ, aparecem alguns termos próprios:

```text
Producer -> Exchange -> Queue -> Consumer
```

A exchange recebe a mensagem do producer e decide para qual fila ela deve ir.

A queue guarda a mensagem até que um consumer processe.

Entre a exchange e a queue existe uma ligação chamada binding.

O binding é como uma regra de entrega:

```text
Exchange -> Binding -> Queue
```

Ele diz para a exchange quais mensagens devem cair em quais filas.

### Routing key, binding key e binding

A routing key é uma informação enviada junto com a mensagem.

Pense nela como uma etiqueta colada no pacote.

Exemplo:

```text
routing key: pedido.criado
```

A binding key é a regra configurada entre uma exchange e uma fila.

Pense nela como o critério que a fila usa para dizer:

```text
quero receber mensagens com esta chave
```

Exemplo:

```text
binding key da fila de notificações: pedido.criado
```

O binding é a ligação completa entre exchange, fila e regra.

Exemplo:

```text
orders.exchange -- binding key pedido.criado --> notifications.queue
```

Quando o producer publica uma mensagem com routing key `pedido.criado`, a exchange compara essa routing key com as binding keys configuradas.

Se bater, a mensagem vai para a fila.

```text
mensagem com routing key pedido.criado
   |
   v
orders.exchange
   |
   | binding key pedido.criado
   v
notifications.queue
```

Alguns exemplos de routing key:

```text
pedido.criado
pedido.cancelado
pagamento.aprovado
```

Em uma exchange do tipo `topic`, a binding key pode usar padrões:

```text
pedido.*       recebe qualquer evento que comece com pedido.
pedido.criado  recebe apenas pedido.criado
```

Agora vale olhar rapidamente para os três tipos mais comuns de exchange.

#### Direct exchange

A `direct exchange` entrega a mensagem para a fila cuja binding key bate exatamente com a routing key.

Use quando o destino é bem específico.

Exemplo:

```text
routing key: pedido.criado

pedido.criado -> fila de pedidos criados
pedido.cancelado -> fila de pedidos cancelados
```

Se a mensagem vem com `pedido.criado`, ela vai para a fila ligada exatamente a `pedido.criado`.

#### Fanout exchange

A `fanout exchange` ignora a routing key e entrega a mensagem para todas as filas ligadas nela.

Use quando todo mundo precisa receber a mesma mensagem.

Exemplo:

```text
pedido.criado
   |
   +-> fila de notificações
   +-> fila de faturamento
   +-> fila de analytics
```

Esse modelo parece um aviso geral. O producer publica uma vez, e todos os consumidores interessados recebem uma cópia em suas próprias filas.

#### Topic exchange

A `topic exchange` usa padrões na routing key.

Use quando você quer roteamento flexível por assunto.

Exemplo:

```text
pedido.*          recebe pedido.criado e pedido.cancelado
pagamento.*       recebe pagamento.aprovado e pagamento.recusado
*.criado          recebe pedido.criado, usuario.criado e produto.criado
```

Esse modelo é muito útil quando os eventos seguem uma convenção de nomes. Em vez de criar uma regra exata para cada evento, você cria padrões.

RabbitMQ costuma ser uma ótima escolha quando o sistema precisa de filas claras, roteamento flexível e processamento de mensagens como tarefas.

### Kafka

Kafka é muito usado quando o sistema trabalha com alto volume de eventos, streaming de dados e histórico de mensagens.

Ele aparece bastante em cenários como:

* eventos de pedidos, pagamentos e acessos;
* integração entre muitos serviços;
* pipelines de dados;
* processamento em tempo real;
* auditoria e reconstrução de histórico;
* analytics.

No Kafka, o desenho mental muda um pouco:

```text
Producer -> Topic -> Consumer Group -> Consumer
```

O producer publica eventos em um tópico.

O tópico é dividido em partitions. Essas partitions ajudam a distribuir carga e escalar o processamento.

Um consumer group representa um grupo de consumers trabalhando juntos. Dentro do mesmo grupo, cada evento é processado por uma instância. Em grupos diferentes, o mesmo evento pode ser lido por aplicações diferentes.

Kafka costuma manter os eventos por tempo ou tamanho configurado, mesmo depois de lidos. Isso permite reprocessamento, leitura histórica e integração com ferramentas de dados.

Kafka não usa exchanges como o RabbitMQ. Os padrões aparecem em cima de tópicos, partitions e consumer groups.

#### Um consumer group para dividir trabalho

Quando várias instâncias usam o mesmo consumer group, elas dividem o processamento.

Exemplo:

```text
tópico pedidos
   |
   +-> payments-service instancia 1
   +-> payments-service instancia 2
```

As duas instâncias fazem parte do mesmo grupo. Kafka distribui as partitions entre elas para aumentar a capacidade de processamento.

Use quando você quer escalar um serviço que faz o mesmo trabalho.

#### Consumer groups diferentes para broadcast

Quando serviços diferentes usam consumer groups diferentes, cada grupo pode receber os mesmos eventos.

Exemplo:

```text
tópico pedidos
   |
   +-> grupo notificações
   +-> grupo faturamento
   +-> grupo analytics
```

Isso lembra o pub/sub: o evento `pedido.criado` é publicado uma vez, mas vários serviços conseguem reagir.

Use quando vários serviços precisam saber que o mesmo evento aconteceu.

#### Partitions para escala e ordem

Partitions dividem um tópico em partes menores.

Exemplo:

```text
tópico pedidos
   |
   +-> partition 0
   +-> partition 1
   +-> partition 2
```

Isso ajuda Kafka a processar alto volume de eventos.

Um cuidado importante: Kafka garante ordem dentro de uma partition, não necessariamente no tópico inteiro.

Se todos os eventos do mesmo pedido precisam manter ordem, é comum usar o `pedidoId` como chave da mensagem. Assim, eventos do mesmo pedido tendem a cair na mesma partition.

Kafka costuma ser uma ótima escolha quando o sistema precisa lidar com muitos eventos, histórico, replay e processamento contínuo de dados.

### RabbitMQ e Kafka resolvem o mesmo problema?

Eles resolvem problemas próximos, mas não são iguais.

Os dois ajudam serviços a se comunicarem por mensagens. Os dois permitem desacoplamento. Os dois aparecem em arquiteturas de microsserviços.

Mas o modelo mental é diferente:

| Ponto | RabbitMQ | Kafka |
| --- | --- | --- |
| Ideia principal | Broker de mensagens com filas e roteamento | Plataforma de eventos baseada em tópicos e log |
| Uso comum | Filas de trabalho, processamento assíncrono, roteamento flexível | Streaming de eventos, alto volume, histórico e replay |
| Unidade central | Exchange e queue | Topic e partition |
| Consumo | Mensagem normalmente sai da fila após confirmação | Evento permanece no tópico durante a retenção configurada |
| Reprocessamento | Normalmente depende de reenvio, dead letter ou nova publicação | É comum reler eventos a partir de offsets |
| Quando costuma encaixar bem | Tarefas assíncronas e integração entre serviços com filas | Eventos em grande escala, dados em tempo real e histórico |

Não existe uma resposta universal.

Se você precisa enviar tarefas para serem processadas por workers, RabbitMQ costuma ser mais direto.

Se você precisa manter um fluxo contínuo de eventos e permitir que vários sistemas leiam ou releiam esse histórico, Kafka costuma fazer mais sentido.

Em projetos reais, a escolha depende de volume, necessidade de replay, modelo de entrega, operação, equipe, infraestrutura e simplicidade.

## Setup inicial

Esta aula é conceitual, então você não precisa instalar nada agora.

Na prática seguinte, vamos usar RabbitMQ com Spring AMQP porque ele deixa bem visível a ideia de producer, exchange, queue, routing key e consumer.

Mesmo assim, leve a ideia principal para qualquer ferramenta:

```text
aplicação publica mensagem -> broker armazena/roteia -> outra aplicação consome
```

Quando o projeto usa Kafka, a prática muda para Spring for Apache Kafka, tópicos, partitions, consumer groups e offsets.

Quando o projeto usa RabbitMQ, a prática muda para Spring AMQP, exchanges, queues, bindings e routing keys.

## Passo a passo

### 1. Entender o problema do HTTP direto

Imagine este código dentro de um service de pedidos:

```text
criar pedido
salvar pedido
chamar serviço de e-mail
chamar serviço de estoque
chamar serviço de faturamento
responder cliente
```

Esse desenho parece simples, mas cria acoplamento.

A API de pedidos passa a depender de vários serviços para responder uma única requisição.

Se a notificação falhar, o pedido talvez nem seja criado. Se o faturamento demorar, o cliente espera mais. Se o estoque mudar o endpoint, a API de pedidos precisa mudar junto.

HTTP continua sendo ótimo para consulta direta e resposta imediata. O problema aparece quando transformamos toda comunicação entre serviços em chamada síncrona.

### 2. Trocar chamada direta por evento

Com mensageria, a API de pedidos faz o trabalho principal:

```text
criar pedido
salvar pedido
publicar pedido.criado
responder cliente
```

Depois disso, outros serviços reagem:

```text
notificações consome pedido.criado
faturamento consome pedido.criado
analytics consome pedido.criado
```

O pedido não precisa conhecer esses serviços diretamente.

Isso é desacoplamento na prática: um serviço publica algo que aconteceu, e outros serviços decidem como reagir.

### 3. Entender entrega assíncrona

Assíncrono significa que uma parte do trabalho fica para depois.

Quando a API publica o evento, ela não espera todos os consumers terminarem.

Isso melhora o tempo de resposta da API, mas traz um cuidado: o sistema passa a trabalhar com consistência eventual.

Consistência eventual quer dizer que nem tudo fica atualizado no mesmo milissegundo.

O pedido pode ser criado agora, e a notificação pode ser enviada alguns segundos depois.

Isso não é necessariamente um problema. Em muitos sistemas, é exatamente o comportamento desejado.

O cliente precisa saber que o pedido foi criado. O e-mail pode chegar logo depois.

### 4. Escolher entre fila e tópico

Agora pense no tipo de trabalho.

Se você tem uma tarefa que deve ser processada por uma instância disponível, uma fila costuma fazer sentido:

```text
gerar nota fiscal -> fila fiscal -> um worker processa
```

Se você tem um evento que vários sistemas precisam conhecer, um tópico costuma fazer mais sentido:

```text
pedido.criado -> tópico pedidos -> vários consumidores interessados
```

RabbitMQ também consegue trabalhar com publicação para vários consumidores usando exchanges e filas diferentes.

Kafka também consegue distribuir processamento entre instâncias usando consumer groups.

Por isso, não escolha a ferramenta só pelo nome do padrão. Olhe para o comportamento que você precisa.

### 5. Pensar em idempotência

Em sistemas com mensageria, uma mensagem pode ser entregue mais de uma vez.

Isso não significa que o broker está errado. Muitas arquiteturas preferem correr o risco de processar novamente a perder uma mensagem importante.

Por isso, consumers devem ser pensados para serem idempotentes.

Idempotente significa que executar a mesma ação mais de uma vez não deve causar problema.

Exemplo simples:

```text
Se o e-mail do pedido 123 já foi enviado, não envie de novo.
```

Outro exemplo:

```text
Se a nota fiscal do pedido 123 já existe, não gere outra nota.
```

Na primeira prática, vamos apenas registrar o evento no log para entender a mecânica. Depois, quando o assunto amadurecer, esse cuidado entra no desenho da solução.

### 6. Entender retry e dead letter

Retry é tentar processar a mensagem novamente depois de uma falha.

Isso ajuda quando o erro é temporário:

```text
serviço externo fora do ar
banco demorou para responder
timeout em uma integração
```

Mas retry infinito é perigoso. Se a mensagem tem dados inválidos, tentar para sempre só ocupa recurso e trava o processamento.

Por isso existe a ideia de dead letter.

Dead letter é um destino para mensagens que não conseguiram ser processadas depois de algumas tentativas.

Em RabbitMQ, é comum usar uma dead letter queue.

Em Kafka, é comum usar um tópico separado para mensagens com erro, muitas vezes chamado de dead letter topic.

O nome muda, mas a ideia é parecida: separar mensagens problemáticas para análise, correção ou reprocessamento controlado.

### 7. Entender que mensageria não substitui API REST

Mensageria complementa HTTP.

Use HTTP quando você precisa de resposta imediata:

```text
buscar produto por id
validar login
criar pedido e devolver o id gerado
```

Use mensageria quando um serviço precisa avisar que algo aconteceu e outros serviços podem reagir depois:

```text
pedido.criado
pagamento.aprovado
produto.sem_estoque
usuario.cadastrado
```

Um sistema real normalmente usa os dois.

API REST cuida bem de comandos e consultas diretas.

Mensageria cuida bem de eventos, integração assíncrona e processamento desacoplado.

## Código completo

Nesta aula conceitual ainda não há código Java completo.

O código completo entra no próximo arquivo da aula 5:

* `Tutorial Mensageria`: cria um producer e um consumer usando RabbitMQ, Spring Boot e Spring AMQP.

Esse tutorial usa RabbitMQ porque é uma boa ferramenta para enxergar filas, exchanges, bindings e consumers logo no começo.

Uma versão com Kafka usaria outros componentes:

* `spring-kafka` no lugar de `spring-boot-starter-amqp`;
* tópicos no lugar de exchanges e filas;
* consumer groups no lugar de consumers ligados a uma fila;
* offsets no lugar da confirmação tradicional de fila.

O conceito principal continua o mesmo: publicar eventos e permitir que outros serviços reajam sem acoplamento direto.

## Erros comuns

* Achar que message broker é sinônimo de RabbitMQ. RabbitMQ é uma ferramenta. Kafka, ActiveMQ, SQS e Google Pub/Sub também aparecem nesse tipo de conversa.
* Achar que Kafka e RabbitMQ são iguais. Eles resolvem problemas próximos, mas usam modelos diferentes.
* Chamar tudo de fila. Em Kafka, o termo central é tópico. Em RabbitMQ, filas são centrais, mas exchanges e bindings também fazem parte do desenho.
* Usar evento como comando direto. `pedido.criado` é melhor do que `enviar.email`, porque o producer não deve mandar no trabalho interno do consumer.
* Achar que mensageria substitui HTTP. Ela complementa. Consultas e respostas imediatas continuam fazendo sentido com REST.
* Ignorar duplicidade. Consumers precisam ser preparados para receber a mesma mensagem mais de uma vez em cenários reais.
* Ignorar observabilidade. Mensageria sem logs, métricas e rastreio de mensagens fica difícil de diagnosticar.
* Colocar regra de negócio no controller. Mesmo com mensageria, controller recebe HTTP e delega para service.

## Resumo

Message broker é um intermediário para comunicação por mensagens.

Mensageria ajuda microsserviços a ficarem menos acoplados.

Producer publica mensagens. Consumer recebe e processa.

Eventos descrevem algo que aconteceu, como `pedido.criado`.

RabbitMQ costuma encaixar bem em filas, roteamento e tarefas assíncronas.

Kafka costuma encaixar bem em streaming de eventos, alto volume, histórico e replay.

Use mensageria quando um serviço precisa avisar que algo aconteceu e outros serviços podem reagir depois.

Use HTTP quando a resposta precisa ser imediata e direta.

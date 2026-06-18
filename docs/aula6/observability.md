# Observabilidade em APIs e Microsserviços com Spring Boot

## Objetivo da aula

Nesta aula, vamos entender o que é observabilidade e por que ela se torna tão importante quando uma aplicação deixa de ser um projeto pequeno e passa a funcionar como uma API real.

A ideia aqui ainda não é configurar ferramentas nem escrever código. Antes disso, faz sentido entender o problema que a observabilidade resolve.

Ao final da aula, você deve conseguir explicar:

- o que é monitoramento;
- o que é observabilidade;
- qual é a diferença entre logs, métricas e traces;
- por que microsserviços precisam de mais visibilidade do que aplicações simples;
- onde entram Spring Boot Actuator, Micrometer e Prometheus nesse assunto.

## Resultado final

Depois desta aula, você terá uma visão clara do papel da observabilidade em uma aplicação Spring Boot.

Você ainda não vai implementar nada aqui. O resultado esperado é conceitual: entender por que uma API precisa expor informações de saúde, registrar eventos importantes, medir comportamento e permitir investigação quando algo dá errado.

Isso prepara o terreno para o tutorial prático, onde a configuração será feita com Spring Boot Actuator, Micrometer e Prometheus.

## Contexto

Quando estamos começando, é comum testar uma API olhando apenas a resposta no navegador, no Postman ou no terminal. Se a requisição retorna certo, parece que está tudo bem. Se dá erro, olhamos a mensagem e tentamos corrigir.

Esse jeito funciona enquanto o sistema é pequeno, está rodando na nossa máquina e tem poucas partes envolvidas.

Em produção, a situação muda.

Uma API pode estar recebendo centenas ou milhares de requisições. Ela pode depender de MongoDB, RabbitMQ, outros microsserviços, API Gateway, serviços externos e containers rodando em máquinas diferentes. Nesse cenário, um erro raramente aparece com uma explicação pronta.

Imagine uma chamada para criar um pedido:

- o cliente envia a requisição para o API Gateway;
- o gateway encaminha para o serviço de pedidos;
- o serviço de pedidos consulta produtos;
- o serviço grava dados no MongoDB;
- depois publica um evento no RabbitMQ;
- outro serviço consome esse evento e continua o processamento.

Se o pedido demora demais, onde está o problema?

Pode ser no gateway, no serviço de pedidos, no banco, na fila, em uma chamada externa ou até em uma regra de negócio mal implementada. Sem observabilidade, a equipe fica tentando adivinhar. Com observabilidade, a investigação passa a se apoiar em dados.

## Explicação conceitual

### Monitoramento

Monitoramento é acompanhar sinais conhecidos do sistema.

Pense no painel de um carro. Ele mostra velocidade, combustível, temperatura e luzes de alerta. Você não precisa abrir o motor para saber que a temperatura subiu. O painel não explica toda a causa do problema, mas avisa que algo merece atenção.

Em uma aplicação, o monitoramento cumpre um papel parecido. Ele acompanha informações como:

- se a aplicação está de pé;
- se o consumo de memória está alto;
- se a CPU está sendo muito usada;
- se a quantidade de erros aumentou;
- se as requisições estão demorando mais do que o normal;
- se uma dependência importante, como o MongoDB, está indisponível.

O monitoramento responde principalmente: o que está acontecendo agora?

Ele é muito útil para alertas. Por exemplo, se a taxa de erro sobe de repente, alguém precisa saber. Se a API parou de responder, isso também precisa aparecer rápido.

O ponto importante é que o monitoramento normalmente trabalha com perguntas que já conhecemos. A equipe define quais sinais quer acompanhar e configura limites aceitáveis.

### Observabilidade

Observabilidade vai além de saber que algo está errado. Ela ajuda a entender por que algo está errado.

Quando uma API fica lenta, o monitoramento pode mostrar que a latência aumentou. Isso é importante, mas ainda não resolve o problema. A pergunta seguinte é mais difícil: por que a latência aumentou?

Pode ter sido uma consulta lenta no MongoDB. Pode ter sido uma fila acumulada no RabbitMQ. Pode ter sido um serviço externo fora do ar. Pode ter sido uma alteração recente no código.

Observabilidade é a capacidade de investigar esse tipo de situação usando dados gerados pelo próprio sistema.

Na prática, uma aplicação observável deixa rastros úteis para a equipe entender seu comportamento. Esses rastros não servem apenas para quando tudo dá errado. Eles também ajudam a melhorar performance, encontrar gargalos e confirmar se uma mudança realmente teve o efeito esperado.

Em microsserviços, isso fica ainda mais importante porque uma requisição pode passar por vários serviços antes de terminar. Se cada serviço for uma caixa fechada, encontrar a causa de um problema vira tentativa e erro.

### Monitoramento e observabilidade não são a mesma coisa

Monitoramento e observabilidade se complementam.

O monitoramento costuma avisar que existe um problema. A observabilidade ajuda a investigar o motivo.

Uma forma simples de pensar é:

- monitoramento mostra o sintoma;
- observabilidade ajuda a encontrar a causa.

Se a API começa a responder em cinco segundos, o monitoramento pode mostrar o aumento da latência. A observabilidade ajuda a descobrir se o tempo foi gasto no controller, no service, no MongoDB, em uma chamada HTTP para outro serviço ou no envio de uma mensagem para o RabbitMQ.

Para sistemas simples, monitorar alguns sinais básicos já ajuda bastante. Para microsserviços, só isso costuma ser pouco, porque o problema pode estar na comunicação entre partes diferentes.

### Os três pilares da observabilidade

A observabilidade costuma ser explicada a partir de três pilares: logs, métricas e traces.

Eles não competem entre si. Cada um responde um tipo de pergunta.

### Logs

Logs são registros de eventos que aconteceram dentro da aplicação.

Quando uma API recebe uma requisição, processa uma regra importante, encontra um erro ou finaliza uma operação relevante, ela pode registrar isso em log.

Um bom log ajuda a responder perguntas como:

- qual operação estava sendo executada?
- qual recurso estava envolvido?
- qual erro aconteceu?
- em qual ponto da aplicação o problema apareceu?
- existe algum identificador que ajude a relacionar uma requisição com outra informação?

Em Java com Spring Boot, logs normalmente são feitos com SLF4J junto com a implementação de logging configurada pelo próprio Spring Boot. O ponto principal para esta aula não é a ferramenta em si, mas a ideia: log não deve ser um texto jogado de qualquer jeito no console. Ele precisa ter contexto.

Um log ruim diz apenas que deu erro.

Um log útil mostra onde o erro aconteceu, qual operação estava em andamento e qual informação ajuda a investigar sem expor dados sensíveis.

Logs são ótimos para investigar situações específicas, mas não são a melhor ferramenta para enxergar tendências gerais. Para isso, usamos métricas.

### Métricas

Métricas são números coletados ao longo do tempo.

Elas ajudam a enxergar o comportamento geral da aplicação. Em vez de olhar uma requisição isolada, você passa a observar padrões.

Exemplos de perguntas que métricas ajudam a responder:

- quantas requisições a API recebe por minuto?
- quanto tempo as requisições estão levando?
- qual endpoint tem mais erro?
- a aplicação está consumindo muita memória?
- o número de conexões com o banco aumentou?
- a fila de mensagens está crescendo?

Métricas são a base de muitos dashboards e alertas. Quando alguém olha um painel com gráficos de latência, quantidade de erros e uso de recursos, está olhando métricas.

No ecossistema Spring Boot, o Micrometer aparece como uma camada de métricas. Ele permite que a aplicação exponha dados em um formato que ferramentas como Prometheus conseguem coletar.

O Prometheus, por sua vez, coleta essas métricas em intervalos definidos e permite consultar o histórico. Mais adiante, essas informações podem aparecer em dashboards, geralmente com ferramentas como Grafana.

O importante por enquanto é entender o papel de cada um:

- Micrometer ajuda a aplicação Spring Boot a produzir métricas;
- Prometheus coleta e armazena essas métricas;
- dashboards transformam esses dados em visualização;
- alertas avisam quando algum comportamento sai do esperado.

### Traces

Traces mostram o caminho de uma requisição pelo sistema.

Esse conceito fica mais claro em microsserviços. Uma chamada pode começar no API Gateway, passar por um serviço de pedidos, consultar um serviço de produtos, acessar o MongoDB e publicar uma mensagem no RabbitMQ.

Se essa chamada demora, olhar apenas o tempo total não basta. Precisamos saber onde o tempo foi gasto.

O trace ajuda justamente nisso. Ele mostra as etapas da requisição e quanto tempo cada parte levou.

Com traces, a equipe consegue responder perguntas como:

- por quais serviços a requisição passou?
- qual serviço demorou mais?
- houve erro em alguma etapa intermediária?
- a lentidão está no banco, em uma chamada HTTP ou em uma fila?
- a mesma requisição gerou eventos em outros serviços?

Traces são especialmente úteis quando o sistema tem várias aplicações conversando entre si. Em uma API pequena, talvez eles não sejam a primeira necessidade. Em uma arquitetura com microsserviços, eles passam a ter muito valor.

### Health check

Health check é uma verificação de saúde da aplicação.

Em Spring Boot, esse assunto aparece com frequência por causa do Spring Boot Actuator. Ele permite expor informações operacionais da aplicação, como saúde, métricas e outros dados úteis para ambiente de execução.

Um health check básico responde se a aplicação está viva. Um health check mais completo pode considerar dependências importantes, como banco de dados, mensageria ou serviços externos.

Isso é importante porque uma aplicação pode estar com o processo em execução, mas ainda assim não conseguir trabalhar corretamente.

Por exemplo:

- a API está de pé, mas perdeu conexão com o MongoDB;
- o serviço responde, mas não consegue publicar mensagens no RabbitMQ;
- o container iniciou, mas a aplicação ainda não está pronta para receber requisições.

Ferramentas de infraestrutura usam health checks para decidir se uma aplicação pode receber tráfego, se precisa ser reiniciada ou se deve ser removida temporariamente do balanceamento.

Para quem está aprendendo, a ideia principal é simples: health check é uma forma padronizada de perguntar para a aplicação se ela está em condições de funcionar.

### Logs, métricas e traces trabalhando juntos

Logs, métricas e traces ficam mais fortes quando são usados em conjunto.

Imagine que uma API de pedidos começou a apresentar erro.

As métricas mostram que a taxa de erro subiu. Isso chama atenção rapidamente.

Os traces mostram que a maior parte das falhas acontece quando o serviço de pedidos tenta consultar o serviço de produtos.

Os logs mostram mensagens de timeout nessa chamada.

Agora a investigação fica muito mais objetiva. Em vez de procurar em todos os serviços, a equipe já sabe onde concentrar a análise.

Esse é o valor da observabilidade: reduzir chute e aumentar clareza.

### Observabilidade no mundo Spring Boot

Em aplicações Spring Boot, alguns nomes aparecem bastante quando falamos de observabilidade.

Spring Boot Actuator expõe informações operacionais da aplicação. Ele é muito usado para health checks e métricas básicas.

Micrometer funciona como uma ponte para métricas. Ele coleta e organiza medidas da aplicação de uma forma compatível com ferramentas externas.

Prometheus coleta métricas expostas pela aplicação e guarda o histórico. Ele é muito usado em ambientes com containers e microsserviços.

Grafana costuma ser usado para montar dashboards em cima dessas métricas.

OpenTelemetry aparece quando o assunto avança para tracing e padronização da coleta de dados de observabilidade.

Nesta disciplina, faz sentido começar pelo básico: health checks, logs e métricas. Depois disso, tracing fica mais fácil de entender, porque você já sabe o problema que ele resolve.

### O que observar em uma API

Nem tudo precisa virar métrica, log ou alerta.

Observabilidade boa não é coletar tudo. É coletar o que ajuda a entender o sistema.

Em uma API REST, alguns sinais costumam ser importantes:

- quantidade de requisições;
- tempo de resposta;
- taxa de erro;
- endpoints mais acessados;
- status HTTP retornados;
- consumo de memória;
- uso de CPU;
- disponibilidade do MongoDB;
- quantidade de mensagens em filas;
- erros de comunicação com outros serviços.

Também é importante observar regras do domínio.

Em uma API de pedidos, por exemplo, pode fazer sentido acompanhar pedidos criados, pedidos cancelados, falhas de pagamento e eventos publicados. Essas informações aproximam a observabilidade da realidade do negócio.

Esse cuidado evita um erro comum: olhar só para CPU e memória e esquecer que a aplicação existe para executar regras importantes.

## Setup inicial

Esta aula não precisa de instalação nem criação de projeto.

Antes de configurar ferramentas, precisamos entender o vocabulário. No tutorial prático, vamos aplicar esses conceitos em uma aplicação Spring Boot usando Actuator, Micrometer e Prometheus.

## Passo a passo

### 1. Comece pelo problema

Antes de pensar em ferramenta, pense no problema.

Uma API em produção precisa responder perguntas simples:

- ela está funcionando?
- está lenta?
- está gerando muitos erros?
- depende de algum serviço que caiu?
- qual parte da requisição está demorando?
- o problema afeta todos os usuários ou apenas uma operação?

Sem essas respostas, qualquer falha vira investigação manual.

### 2. Entenda o papel do monitoramento

Monitoramento acompanha sinais conhecidos.

Ele serve para avisar rapidamente quando algo saiu do comportamento esperado. Isso inclui aplicação fora do ar, erro demais, lentidão ou consumo exagerado de recursos.

Use monitoramento para enxergar o estado atual do sistema.

### 3. Entenda o papel da observabilidade

Observabilidade ajuda a investigar.

Ela entra quando o alerta apareceu e agora precisamos entender a causa. A investigação usa logs, métricas e traces para montar uma visão mais completa.

Use observabilidade para descobrir por que o sistema se comportou daquele jeito.

### 4. Separe logs, métricas e traces

Logs mostram eventos.

Métricas mostram números ao longo do tempo.

Traces mostram o caminho de uma requisição.

Quando esses três tipos de informação são bem usados, a equipe consegue investigar problemas com muito mais precisão.

### 5. Comece simples

Para uma primeira aplicação Spring Boot, não faz sentido começar com uma pilha enorme de ferramentas.

Um caminho mais didático é:

- primeiro entender logs;
- depois expor health checks;
- depois coletar métricas;
- depois visualizar essas métricas;
- depois estudar tracing.

Essa evolução acompanha a complexidade do sistema. Conforme a aplicação cresce, a observabilidade também cresce.

## Código completo

Esta aula não tem código completo porque o objetivo aqui é teórico.

A implementação fica para o tutorial prático. Lá sim fará sentido configurar dependências, endpoints de saúde, métricas e integração com Prometheus.

## Erros comuns

### Confundir log com observabilidade completa

Ter logs ajuda, mas não resolve tudo.

Logs mostram eventos específicos. Eles não substituem métricas, dashboards, alertas e traces. Uma aplicação pode ter muitos logs e ainda assim ser difícil de investigar.

### Registrar informação demais

Log demais atrapalha.

Quando tudo vira log, encontrar o que importa fica difícil. Além disso, logs em excesso podem aumentar custo de armazenamento e prejudicar desempenho.

O ideal é registrar eventos relevantes, erros importantes e informações que realmente ajudem a investigar.

### Registrar dados sensíveis

Logs não devem expor senha, token, documento, cartão, segredo de API ou informação pessoal desnecessária.

Esse cuidado é técnico e também ético. Log costuma circular por ferramentas externas e pode ficar armazenado por muito tempo.

### Criar alertas para tudo

Alerta demais faz a equipe parar de prestar atenção.

Um bom alerta precisa indicar algo que exige ação. Se um alerta dispara toda hora e ninguém faz nada, ele provavelmente está mal configurado.

### Olhar só infraestrutura

CPU, memória e disco são importantes, mas não contam a história inteira.

Uma API pode estar com CPU baixa e, mesmo assim, falhar em uma regra importante. Por isso, também precisamos observar sinais da aplicação e do negócio.

### Começar por ferramentas antes de entender o conceito

Prometheus, Grafana, Actuator e OpenTelemetry são ferramentas úteis, mas elas não substituem entendimento.

Antes de configurar qualquer coisa, você precisa saber que pergunta quer responder. Ferramenta sem pergunta vira painel bonito que ninguém usa.

## Resumo

Observabilidade é a capacidade de entender o comportamento interno de uma aplicação a partir dos sinais que ela produz.

Monitoramento ajuda a perceber que algo saiu do esperado. Observabilidade ajuda a investigar o motivo.

Os três pilares mais conhecidos são logs, métricas e traces:

- logs registram eventos importantes;
- métricas mostram números ao longo do tempo;
- traces acompanham o caminho de uma requisição entre serviços.

No ecossistema Spring Boot, vamos trabalhar principalmente com Spring Boot Actuator, Micrometer e Prometheus. Primeiro precisamos entender a teoria. Depois, no tutorial, esses conceitos viram configuração e prática.
